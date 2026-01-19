# Networking
## Introduction
This document describes how networking is implemented in the Toro unikernel. The networking functionality is provided by the `Network.pas` unit, which exposes the underlying mechanisms that enable applications to communicate over a network. This is a technical reference document, not a tutorial.

The document is organized as follows: it begins by explaining what network drivers are and how they register with the kernel. Next, it covers the low-level primitives for sending and receiving packets. Finally, it describes how the socket abstraction layer is built on top of these packet primitives to provide high-level networking capabilities.

## Network drivers
Network drivers are units that register a `TNetworkInterface` with the kernel to enable sending and receiving packets over a network. To register a driver, the `RegisterNetworkInterface()` procedure is called with a `PNetworkInterface` structure:
```pascal
procedure RegisterNetworkInterface(NetInterface: PNetworkInterface);
```
During registration, the kernel adds the `PNetworkInterface` structure to a list containing all registered network drivers. Note that the kernel does not verify whether a driver has already been registered. The `TNetworkInterface` structure represents a network driver and exposes the following interface:
```pascal
  TNetworkInterface = record
    Name: array[0..MAX_NET_NAME-1] of Char;
    Minor: LongInt;
    CPUID: LongInt;
    Device: Pointer;
    IncomingPacketsTail: PPacket;
    IncomingPackets: PPacket;
    Start: procedure (NetInterface: PNetworkInterface);
    Send: procedure (NetInterface: PNetworkInterface;Packet: PPacket);
    Reset: procedure (NetInterface: PNetworkInterface);
    Stop: procedure (NetInterface: PNetworkInterface);
    Next: PNetworkInterface;
  end;

```
The structure contains the following fields:
* `Name`, an array of characters that identifies the driver (e.g., ne2000, virtio-net, e1000)
* `Minor`, a LongInt that identifies a specific device instance. For example, in the IDE disk driver, the minor number identifies one of the four possible disks.
* `CPUID`, a LongInt that identifies the CPU to which the device has been dedicated
* `Device`, an untyped pointer to the driver's internal context, used internally by the driver
* `IncomingPackets`, a queue of incoming packets stored as a simple linked list following a *last in first out* (LIFO) strategy

The structure also defines a set of callback methods that the kernel uses to interact with the driver:
* `start()`, initializes the driver
* `send()`, sends a packet through the driver
* `reset()`, re-initializes the device
* `stop()`, stops the driver from receiving packets

### Driver Dedication
Before a network driver can be used, it must be dedicated to a CPU core. Each device can only be dedicated to a single core for the lifetime of the system, and the `CPUID` field identifies which CPU is allowed to access the driver.

To dedicate a driver to the current core, call `DedicateNetworkSocket()` with the driver's name:
```pascal
procedure DedicateNetworkSocket(const Name: PXChar);
```
This procedure searches for the driver by name in the registered drivers list. If the driver has already been dedicated to another core, the procedure fails. Otherwise, it dedicates the driver to the current core and initializes the network stack by calling `LocalNetworkInit()`:
```pascal
function LocalNetworkInit(PacketHandler: Pointer): Boolean;
```
`LocalNetworkInit()` creates a new thread in the local core that will execute the `PacketHandler()` function. Once the thread is created, the function invokes the `start()` method of the network interface dedicated to the current core to begin operation.

### Packet Processing
The packet handler function, which processes incoming packets, is declared as:
```pascal
function ProcessSocketPacket(Param: Pointer): PtrUInt;
```
Toro currently only supports packets conforming to the **virtio-vsock** protocol. The `ProcessSocketPacket()` function is specifically designed to handle these packet types. To add support for other protocols (such as Ethernet), a different packet handler function would need to be implemented to parse the alternative packet format.

## Network Packets
The Network unit provides two core functions for sending and receiving packets: `SysNetworkSend()` and `SysNetworkRead()`. These functions operate on the network driver that has been registered and dedicated to the local core from which they are called. Note that a network driver must be registered and dedicated before these functions can be used.

### Sending Packets
To send a packet, use the `SysNetworkSend()` function:
```pascal
procedure SysNetworkSend(Packet: PPacket);
```
**Important considerations:**
- This function is neither *thread-safe* nor *interrupt-safe*. It must not be called from an interrupt handler.
- The cooperative scheduler ensures that the current thread is never preempted, which means it's impossible to have two instances of this function executing simultaneously in different threads on the same core.
- When sending a packet, the caller loses ownership of the packet. The network stack takes responsibility for releasing the packet memory after the driver confirms that it has been sent successfully.

### Transmission Path
The packet transmission process follows these steps:

1. **Function call**: When `SysNetworkSend()` is called, the network stack retrieves the network interface dedicated to the current core and calls the driver's `send()` method with the packet.

2. **Driver processing**: The driver processes the packet. For the virtio-vsock driver, this involves enqueueing the `Packet.Data` buffer into the available ring of the transmission queue. The function returns immediately without waiting for the packet to be actually transmitted.

3. **Hardware transmission**: The hardware device consumes the buffer from the transmission queue and sends it over the network.

4. **Completion notification**: Once the device has consumed the buffer and the packet has been sent, the driver notifies the kernel by calling `DequeueOutgoingPacket()`.

5. **Memory cleanup**: The kernel releases the `TPacket` structure and its payload (`Packet.Data`) only after receiving confirmation from the driver that the packet has been successfully sent.

This asynchronous design allows the caller to continue execution without blocking on network transmission. In the case of virtio-vsock, the packet buffer allocated by `SysSocketSend()` remains in use until the device confirms transmission completion.

### Reception Path
Packet reception follows these steps:

1. **Hardware interrupt**: When the hardware receives a new packet, it triggers an interrupt. The network driver handles this interrupt.

2. **Kernel notification**: The driver notifies the kernel by calling `EnqueueIncomingPacket()`, which adds the packet to the incoming packets queue.

3. **Queue management**: Packets are stored in a simple linked list queue associated with the network interface. The queue follows a *last in first out* (LIFO) strategy. The network driver is the sole producer of this queue, while `SysNetworkRead()` is the sole consumer.

4. **Reading packets**: To retrieve packets, call the `SysNetworkRead()` function:
```pascal
function SysNetworkRead: PPacket;
```
This function returns a packet from the `IncomingPackets` queue of the network interface dedicated to the current core. If no packets are available, it returns `nil`.

**Thread safety**: The cooperative scheduler ensures that only one thread executes `SysNetworkRead()` at a time on a given core. However, since the queue can be modified by interrupt handlers (when new packets arrive), `SysNetworkRead()` is not interrupt-safe. The caller must disable interrupts before calling this function to prevent race conditions between packet reception and packet reading.

## Processing Incoming Packets
The `ProcessSocketPacket()` function is the packet handler that processes incoming packets from the virtio-vsock driver. This function runs in a dedicated thread created by `LocalNetworkInit()`. The function implements a continuous loop that reads and processes packets.

### Packet Reading Loop
The function begins by reading packets from the network interface:
```pascal
function ProcessSocketPacket(Param: Pointer): PtrUInt;
var
  Packet: PPacket;
  VPacket: PVirtIOVSocketPacket;
  Service: PNetworkService;
  ClientSocket: PSocket;
  Source, Dest: PByte;
  total: DWORD;
begin
  while True do
  begin
    Packet := SysNetworkRead;
    if Packet = nil then
    begin
      SysSetCoreIdle;
      Continue;
    end;
```

When `SysNetworkRead()` returns `nil`, it means there are no packets available in the queue. In this case:
- `SysSetCoreIdle()` is called to inform the kernel that the core is entering an idle state. This function tracks the time elapsed since the last network interrupt.
- If the idle time exceeds `WAIT_IDLE_CORE_MS`, the core is halted until a new interrupt arrives, saving power.
- While waiting, the function continues calling `SysThreadSwitch()` to allow other threads to execute via the cooperative scheduler.

This design ensures efficient CPU usage by putting the core to sleep when no network activity is occurring, while still allowing other threads to run during idle periods. 


**TODO: However, this function must be used with caution. Networking is asumming that the current core has been dedicated for networking so halting it would not affect other components. This however is not the case if other thread are running in the core doing something different.** 

### Packet Parsing and Processing
Once a packet is successfully read from the queue, it must be parsed and processed according to the virtio-vsock protocol. The packet data is first cast to a `PVirtIOVSocketPacket` structure, which contains a header with operation information and optional payload data.

The processing logic uses a `case` statement based on the operation type (`VPacket.hdr.op`) to determine how to handle each packet. The virtio-vsock protocol defines several operation types, each requiring different processing:

- **VIRTIO_VSOCK_OP_REQUEST**: An incoming connection request from a remote peer. The handler validates the request and creates a new client socket if a listening service exists on the destination port.

- **VIRTIO_VSOCK_OP_RW**: A read/write packet containing data. The handler copies the packet payload into the receiving socket's buffer, respecting window size limits, and updates flow control credits.

- **VIRTIO_VSOCK_OP_SHUTDOWN**: A shutdown request from the remote peer. The handler transitions the socket to a closing state if it's currently transmitting.

- **VIRTIO_VSOCK_OP_CREDIT_UPDATE**: A flow control credit update. The handler updates the remote window count to reflect available buffer space on the sender side.

- **VIRTIO_VSOCK_OP_RST**: A reset packet indicating connection termination or rejection. The handler either cleans up disconnected sockets or marks connecting sockets as blocked.

- **VIRTIO_VSOCK_OP_RESPONSE**: A response to a connection request. The handler updates the socket state to transmitting, indicating the connection has been accepted.

After processing, the packet memory is freed using `ToroFreeMem()`. If a socket cannot be found for a given packet (e.g., no listening service for a REQUEST or no matching client socket for data packets), the handler sends a reset packet back to the sender using `VSocketReset()`.

## Sockets
The socket layer sits on top of the packet primitives (`SysNetworkSend()` / `SysNetworkRead()`) and the packet-processing thread (`ProcessSocketPacket()`). It parses incoming virtio-vsock packets and translates them into stream-oriented connections between two endpoints. Applications use the `SysSocket*` API to create sockets, accept or establish connections, and exchange bytes.

Toro currently supports only the **AF_VSOCK** socket family (virtio-vsock), and only **SOCKET_STREAM** sockets.
In virtio-vsock, endpoints are addressed by (CID, port). Toro uses CID `2` for the host. The guest CID is provided to QEMU and is discovered by the virtio-vsock driver at boot.

### Quick usage example (server and client)
The code below shows the typical control flow when using sockets in Toro. It is intentionally minimal (no error handling, no protocol parsing).

#### Server (listening socket)
```pascal
var
  Server, Client: PSocket;
  Buf: array[0..1023] of Char;
  N: LongInt;
begin
  Server := SysSocket(SOCKET_STREAM);
  Server.SourcePort := 8080;
  Server.Blocking := True;

  if not SysSocketListen(Server, 50) then
    Exit;

  while True do
  begin
    Client := SysSocketAccept(Server); // blocks until a connection arrives

    // Wait for new data (optional, but typical for blocking sockets)
    if SysSocketSelect(Client, 5000) then
    begin
      N := SysSocketRecv(Client, PChar(@Buf[0]), SizeOf(Buf), 0);
      if N > 0 then
        SysSocketSend(Client, PChar(@Buf[0]), N, 0); // echo back
    end;

    SysSocketClose(Client);
  end;
end;
```

#### Client (connecting socket)
```pascal
const
  Msg: PChar = 'hello';
var
  S: PSocket;
  Buf: array[0..1023] of Char;
  N: LongInt;
begin
  S := SysSocket(SOCKET_STREAM);
  S.Blocking := True;

  // Connect to host:port over AF_VSOCK
  S.DestIp := 2;     // host CID
  S.DestPort := 8080;

  if not SysSocketConnect(S) then
    Exit;

  SysSocketSend(S, Msg, 5, 0);

  if SysSocketSelect(S, 5000) then
  begin
    N := SysSocketRecv(S, PChar(@Buf[0]), SizeOf(Buf), 0);
    // ... process reply ...
  end;

  SysSocketClose(S);
end;
```

### Socket structure
Internally, sockets are represented by the `TSocket` record. The same structure is used for both server-side and client-side sockets; fields such as `Mode`, `State`, `SourcePort`, and `DestPort` determine the role and connection state.

At a high level, a `TSocket` stores the local/remote addressing (`SourcePort`, `DestPort`, `DestIp`), the role and lifecycle (`Mode`, `State`, `SocketType`, `Blocking`), and the state needed by the stream implementation: receive buffering (`Buffer`, `BufferReader`, `BufferLength`), flow control (`RemoteWinLen`, `RemoteWinCount`), and wait/dispatch bookkeeping (`DispatcherEvent`, `TimeOut`). For listening sockets, `ConnectionsQueueLen`/`ConnectionsQueueCount` track the pending accept queue, while `UserDefined` is available for application-specific per-socket state and `Next` links sockets in internal lists.

**TODO: In a future version, socket operations should be registered based on the socket family, allowing multiple families to coexist (e.g., inet + vsock) and be handled by different drivers/backends.**

### SysSocket()
A socket is created by calling `SysSocket()`:
```pascal
function SysSocket(SocketType: LongInt): PSocket;
```
The function returns a pointer to a `TSocket` structure or `nil` if it fails. The only supported type is `SOCKET_STREAM` (stream sockets); `SOCKET_DGRAM` is not supported. Toro currently assumes **AF_VSOCK** as the only socket family. The returned socket pointer is used as input to the other `SysSocket*` functions.

### SysSocketListen()

`SysSocketListen()` is typically used after `SysSocket()` to make a socket listen on a local port. The function is defined as follows:
```pascal
function SysSocketListen(Socket: PSocket; QueueLen: LongInt): Boolean;
```
The function accepts `Socket` (a pointer to a previously created socket) and `QueueLen` (the maximum number of pending incoming connection requests, i.e., backlog). When the number of pending requests exceeds `QueueLen`, the kernel starts dropping requests. The function returns `True` on success, or `False` otherwise.

Before calling `SysSocketListen()`, the user must set the local port (`Socket.SourcePort`) and the socket mode (blocking or non-blocking). This is usually done as:
```pascal
  HttpServer := SysSocket(SOCKET_STREAM);
  HttpServer.SourcePort := 80;
  HttpServer.Blocking := True;
  SysSocketListen(HttpServer, 50);
```
In this example, `HttpServer` listens on port 80 with a backlog of 50. Internally, `SysSocketListen()` registers the listening socket in a per-port service table and initializes a queue of pending incoming connections; the queue length is bounded by `QueueLen`.

### SysSocketAccept()
`SysSocketAccept()` returns an incoming connection for a listening socket, i.e., it dequeues a socket from the pending connection queue created by `SysSocketListen()`. The function takes the server (listening) socket as input and returns a new socket representing the accepted connection. If the server socket is blocking, the function waits until a connection arrives; if it is non-blocking, the function returns `nil` when no connection is available.

The function is defined as follows:
```pascal
function SysSocketAccept(Socket: PSocket): PSocket;
```
Internally, incoming connections are enqueued into a per-port list/queue. `SysSocketAccept()` scans that queue and returns the first socket that is ready to be accepted. If no connection is available and the server socket is blocking, it yields with `SysThreadSwitch()` and retries until a connection arrives.

### SysSocketSelect()
`SysSocketSelect()` waits for an event on a socket, e.g., new data, timeout, or connection closed. The behavior depends on whether the socket is blocking: for non-blocking sockets it performs a single check (data available / closed), sets a timeout, and returns immediately; for blocking sockets it loops until an event occurs or the timeout expires, yielding with `SysThreadSwitch()` while waiting. For blocking sockets, the function returns `False` when the timeout expires (or when the socket is closing), and otherwise returns `True`. For non-blocking sockets, the function returns immediately; it sets `Socket.DispatcherEvent` to `DISP_RECEIVE` or `DISP_CLOSE` when an event is already available.

```pascal
function SysSocketSelect(Socket: PSocket; TimeOut: LongInt): Boolean;
```
The function exits when one of the following happens: (1) the remote peer disconnected (`Socket.State = SCK_LOCALCLOSING`), (2) new data is available in the incoming buffer (`Socket.BufferReader < Socket.Buffer + Socket.BufferLength`), or (3) the timeout has expired. If none of these conditions is satisfied, the function calls the scheduler. Note that the timeout is approximate because the scheduler is cooperative: the thread may not be scheduled again immediately when the deadline passes.


The user can use the `UserDefined` field to store application-specific information associated with a socket. For example, the StaticWebServer attaches per-connection request state (buffers, counters, etc.) to the accepted socket via `Socket.UserDefined` and then passes that socket to a worker thread.

### SysSocketSend()
Sending data through a socket is done with `SysSocketSend()`. The function fragments the payload into virtio-vsock packets and enqueues them using `SysNetworkSend()`. The function may yield (call the scheduler) when the remote receive window is full, and continues once the remote side advertises more credit. The function performs one copy from the user's buffer into the packet payload. Packets allocated by this function are released later by the kernel when the driver reports that the device has consumed them. In the current implementation, the function waits/yields until it can enqueue the whole payload, so the return value is typically `0` when it returns.

```pascal
function SysSocketSend(Socket: PSocket; Addr: PChar; AddrLen, Flags: UInt32): LongInt;
```

### SysSocketRecv()
This function reads up to `AddrLen` bytes from the socket receive buffer and copies them into `Addr` (one copy). The function immediately exits when the socket is not in `SCK_TRANSMITTING` state, when `AddrLen = 0`, or when there is no data available to read. The function returns the number of bytes read from the internal buffer and **consumes** that data. If there is no data available, the function returns `0`. This is typically used together with `SysSocketSelect()` to wait for incoming data.
```pascal
function SysSocketRecv(Socket: PSocket; Addr: PChar; AddrLen, Flags: UInt32): LongInt;
```

### SysSocketPeek()
This function behaves similarly to `SysSocketRecv()`, but the data is not consumed from the socket buffer. This is useful to inspect or “probe” the buffer (e.g., to check whether a complete header/structure is available) before consuming it with `SysSocketRecv()`.

```pascal
function SysSocketPeek(Socket: PSocket; Addr: PChar; AddrLen: UInt32): LongInt;
```
### SysSocketConnect()
This function connects a socket to a remote peer. The caller must set `Socket.DestIp` (destination CID) and `Socket.DestPort` before calling it. The function returns `True` if the connection is established (socket reaches `SCK_TRANSMITTING`), or `False` on error/timeout. The function is defined as follows:
```pascal
function SysSocketConnect(Socket: PSocket): Boolean;
```
The function allocates the local receive buffer, allocates a free source port, registers the socket in the local service table, and sends a `VIRTIO_VSOCK_OP_REQUEST` packet to the destination CID/port. It then waits (with a timeout) until the peer accepts the connection and the socket transitions to `SCK_TRANSMITTING`. If the timeout expires or the socket becomes blocked, the function returns `False`.

### SysSocketClose()
`SysSocketClose()` closes a socket. If the socket is already in `SCK_LOCALCLOSING` (the peer already closed), the function releases the socket immediately. Otherwise, the function sends a `VIRTIO_VSOCK_OP_SHUTDOWN` packet to the peer and transitions the socket to `SCK_PEER_DISCONNECTED` while waiting for the peer to acknowledge the close. The socket is released when that confirmation arrives.
```pascal
procedure SysSocketClose(Socket: PSocket);
```
