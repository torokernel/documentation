# Networking
## Introduction
This document presents how networking works in Toro unikernel. The unit in charge of networking is `Network.pas`. This document only describes the machinery to provide networking to the user application. This is not a tutorial. This document first presents what is a network driver and how it is registered to the kernel. Then, the document presents the primitives to send and receive packets. It finishes by presenting how sockets are implemented by relying on those primitives.

## Network drivers
Network drivers are units that register a `TNetWorkInterface` to the kernel. The kernel uses this interface to send and receive packets through a network. A driver calls `RegisterNetworkInterface()` to register a new `PNetworkInterface` structure to the kernel. This procedure is declared as follows:
```pascal
procedure RegisterNetworkInterface(NetInterface: PNetworkInterface);
```
During registration, the kernel enqueues the `PNetWorkInterface` structure into a list that contains all the registered drivers. The kernel does not verify if the driver has been already registered. The `TNetworkInterface` represents a network driver and contains the following information:
```pascal
  TNetworkInterface = record
    Name: array[0..MAX_NET_NAME-1] of Char;
    Minor: LongInt;
	CPUID: LongInt;
    Device: Pointer;
    IncomingPacketsTail: PPacket;
    IncomingPackets: PPacket;
    Start: procedure (NetInterface: PNetworkInterface);
    Send: procedure (NetInterface: PNetWorkInterface;Packet: PPacket);
    Reset: procedure (NetInterface: PNetWorkInterface);
    Stop: procedure (NetInterface: PNetworkInterface);
    Next: PNetworkInterface;
  end;

```
* `Name`, an array of chars that identifies the driver, e.g., ne2000, virtio-net, e1000
* `Minor`, a LongInt that identifies a single device. For example, in the Ide disk driver, the minor identifies one of the possible four disks.
* `Device`, a pointer to the context for the driver. This pointer is not typed and it is used internally by the driver.
* `IncomingPackets`, a queue of incoming packets that follows a *last in first out* strategy

The structure defines a set of methods to interact with the kernel:
* `start()`, initialises the driver
* `send()`, sends a packet through the driver.
* `reset()`, tells the driver to re-initialize the device
* `stop()`, tells the driver to stop receiving packets.

Each network driver must be dedicated to a core before the user can use it. The device can only be dedicated to a single core during the lifetime of the system. In other words, the dedicated core cannot change. The `CPUID` identifies which CPU is allowed to access a driver.

The user calls `DedicateNetworkSocket()` to dedicate a driver to the current core. The procedure uses the name of the driver to select a driver from the list. The procedure is declared as follows:
```pascal
procedure DedicateNetworkSocket(const Name: PXChar);
```
This procedure fails if the driver has been already dedicated to a core, if not, it dedicates the driver to current core, and then, initializes the network stack by invoking the `LocalNetworkInit()` procedure:
```pascal
function LocalNetworkInit(PacketHandler: Pointer): Boolean;
```
This function creates a new thread that ends up invoking the `PacketHandler()` function. After the thread is created, `LocalNetworkInit()` invokes the `start()` function of the network card that has been dedicated to the current core.

The function that handles packets is declared as follows:
```pascal
function ProcessSocketPacket(Param: Pointer): PtrUInt;
```
Toro only supports packets that conforms to **virtio-vsock**. Thus, the `ProcessSocketPacket()` function expects those types of packets. To add support for ethernet packets, a different function shall be used.

## Network Packets
* The Network unit provides the `SysNetworkSend()` and `SysNetworkRead()` functions to send and receive network packets. The functions use the driver that has been registered in the current core. These functions require you to first register a network card.
* `SysNetWorkPacketSend()` is defined as follows:
```pascal
procedure SysNetworkSend(Packet: PPacket);
```
* This function is neither thread safe nor interruption safe. For this reason, it must not be used in the context of an interruption handler. The cooperative scheduler ensures non-preemption. That said, it is impossible to have two instances of the function in a different thread in the same core.
* When using this function, the sender loses the ownership of the packet. It is the network stack that releases the packet after the driver confirms that it has been sent.

### Transmission Path
* The transmission of a packet follows the these steps:
1. During the SysNetworkSend(), the network stack gets the network interface that is dedicated to the current core and issues a `send()` in the corresponding driver. The function returns without knowing if the packet has been sent.
2. The driver handles the new packet. For example, in the case of `VirtIOVSocket`, the `Packet.Data` buffer is enqueued in the available ring of the transmission queue.
3. Once the packet has been sent, i.e., the device has consumed the buffer, the driver tells the kernel that the packet has been sent by invoking `DequeueOutgoingPacket()`:
4. The kernel releases the `TPacket` structure and its payload, i.e., `Packet.Data`.

Note that in the case of the virtio-vsock driver, the data is enqueued in the available ring of the transmission queue. This is the buffer that SysSocketSend has allocated. After the device consumes the buffer, the driver informs the kernel that the packet has been sent. The kernel can release the memory for the packet.

### Reception Path
* The reception of a packet happens in these steps:
1. When a new packet is received, the driver informs the kernel by using `EnqueueIncomingPacket()`.
2. The packet is enqueued in the network's card incoming packet queue. The only producer is the network card and the only consumer is the `SysNetworkRead()` function which is usually invoked by a single thread from the same core. Together with the cooperative scheduler, this ensures no race condition when reading such a list. This function is not interrupt-safe so the caller must disable interruptions. Otherwise, the arrival of a new packet will race with the current function. Packets are read by using the `SysNetworkRead()` function. This function returns a packet from the `IncomingPackets` queue of the current core.

## Processing Incoming Packets
TODO:
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
  {$IFDEF FPC} Result := 0; {$ENDIF}
  while True do
  begin
    Packet := SysNetworkRead;
    if Packet = nil then
    begin
      SysSetCoreIdle;
      Continue;
    end;
    VPacket := Packet.Data;
    case VPacket.hdr.op of
      VIRTIO_VSOCK_OP_REQUEST:
        begin
          Service := VSocketValidateIncomingConnection(VPacket.hdr.tp, VPacket.hdr.dst_port);
          if Service = nil then
            VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port)
          else
          begin
            ClientSocket := VSocketInit();
            if ClientSocket <> nil then
            begin
              Inc(Service.ServerSocket.ConnectionsQueueCount);
              ClientSocket.Next := Service.ClientSocket;
              Service.ClientSocket := ClientSocket;
              ClientSocket.SourcePort := VPacket.hdr.dst_port;
              ClientSocket.DestPort := VPacket.hdr.src_port;
              ClientSocket.DestIp := VPacket.hdr.src_cid;
              ClientSocket.RemoteWinLen := VPacket.hdr.buf_alloc;
              ClientSocket.RemoteWinCount := VPacket.hdr.fwd_cnt;
              ClientSocket.Blocking := Service.ServerSocket.Blocking;
              VSocketResponse(ClientSocket);
            end else
              VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
          end;
        end;
      VIRTIO_VSOCK_OP_RW:
        begin
          ClientSocket := VSocketGetClient(VPacket.hdr.dst_port, VPacket.hdr.src_port);
          if ClientSocket <> nil then
          begin
            Source := Pointer(PtrUInt(@VPacket.data));
            Dest := Pointer(PtrUInt(ClientSocket.Buffer)+ClientSocket.BufferLength);
            if  ClientSocket.BufferLength + VPacket.hdr.len > MAX_WINDOW then
              total := MAX_WINDOW - ClientSocket.BufferLength
            else
              total :=  VPacket.hdr.len;
            Move(Source^, Dest^, total);
            Inc(ClientSocket.BufferLength, total);
            VSocketUpdateCredit(ClientSocket);
          end else
            VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
        end;
      VIRTIO_VSOCK_OP_SHUTDOWN:
        begin
          ClientSocket := VSocketGetClient(VPacket.hdr.dst_port, VPacket.hdr.src_port);
          if ClientSocket <> nil then
          begin
            If ClientSocket.State = SCK_TRANSMITTING then
            begin
              ClientSocket.State := SCK_LOCALCLOSING;
              VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
            end;
          end else
            VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
        end;
      VIRTIO_VSOCK_OP_CREDIT_UPDATE:
        begin
          ClientSocket := VSocketGetClient(VPacket.hdr.dst_port, VPacket.hdr.src_port);
          if ClientSocket <> nil then
            ClientSocket.RemoteWinCount := VPacket.hdr.fwd_cnt
          else
            VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
        end;
      VIRTIO_VSOCK_OP_RST:
        begin
          ClientSocket := VSocketGetClient(VPacket.hdr.dst_port, VPacket.hdr.src_port);
          if ClientSocket <> nil then
          begin
            If ClientSocket.State = SCK_PEER_DISCONNECTED then
              FreeSocket(ClientSocket)
            else if ClientSocket.State = SCK_CONNECTING then
              ClientSocket.State := SCK_BLOCKED;
          end else
              VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
        end;
      VIRTIO_VSOCK_OP_RESPONSE:
        begin
          Service := GetNetwork.SocketStream[VPacket.hdr.dst_port];
          if Service = nil then
          begin
            VSocketReset(VPacket.hdr.src_cid, VPacket.hdr.src_port, VPacket.hdr.dst_port);
          end else
          begin
            ClientSocket := Service.ClientSocket;
            ClientSocket.RemoteWinLen := VPacket.hdr.buf_alloc;
            ClientSocket.RemoteWinCount := VPacket.hdr.fwd_cnt;
            ClientSocket.State := SCK_TRANSMITTING;
          end;
        end;
    end;
    ToroFreeMem(Packet);
  end;
end;
```

## Sockets
* On top of the networking primitives to send and receive packets, there is the socket layer that parses those packets and translates them into stream or datagram connections between two sockets. Sockets allow applications to communicate over a network. Toro supports only AF_VSOCK sockets. The Toro API to manipulate sockets looks like the POSIX API.
* When using AF_VSOCK, the host is always identified with the number 2 and the guest id is defined by the user when invoking QEMU.
*
* A socket is defined as follows:
```pascal
TSocket = record
  SourcePort, DestPort: UInt32;
  DestIp: TIPAddress;
  SocketType: LongInt;
  Mode: LongInt;
  State: LongInt;
  RemoteWinLen: UInt32;
  RemoteWinCount: UInt32;
  BufferReader: PChar;
  BufferLength: UInt32;
  Buffer: PChar;
  ConnectionsQueueLen: LongInt;
  ConnectionsQueueCount: LongInt;
  NeedFreePort: Boolean;
  DispatcherEvent: LongInt;
  TimeOut: Int64;
  BufferSenderTail: PBufferSender;
  BufferSender: PBufferSender;
  UserDefined: Pointer;
  Blocking: Boolean;
  Next: PSocket;
end;
```
* The network stack maps this structure to a `AF_VSOCK` structure.
* TODO: add state-machine describing sockets

### SysSocket()
* The user instantias a new socket by using the `SysSocket()` function:
```pascal
function SysSocket(SocketType: LongInt): PSocket;
```
* This function returns a pointer to a TSocket structure or Nil if it fails.
* The only supported type is SOCKET_STREAM which refers to stream sockets.
* A SOCKET_STREAM has control flow. SOCKET_DGRAM is not supported.
* The function simply initialises the structures and returns a new socket.
* The returned pointer to a socket is used for the remainder functions.
* For example, the SysSocketListen() is often used after SysSocket() to bind a socket to a port. This function is defined as follows:
```pascal
function SysSocketListen(Socket: PSocket; QueueLen: LongInt): Boolean;
```
* This function accepts Socket and QueueLen as arguments. Socket is a pointer to a previously created socket. QueueLen is the number of requests that the socket can accept at the same time. When the number of requests is over QueueLen, the kernel starts to drop the requests.
* This function returns True if success, Or, False, otherwise.
* Before calling this function, the user has to set the local port in which the socket will listen and the mode, i.e., blocking or non-blocking socket. This is usually done as:
```pascal
  HttpServer := SysSocket(SOCKET_STREAM);
  HttpServer.Sourceport := 80;
  HttpServer.Blocking := True;
  SysSocketListen(HttpServer, 50);
```
* In this case, the HttpServer socket listens at port 80 and the length of the queue is 50.
* SysSocketListen() only binds the port to the socket. It is SysSocketAccept() to get incoming sockets connections.
* SysSocketAccept() accepts a pointer to a Socket as an argument. The function waits until a new connection arrives. When this happens, it returns the new socket.
* The function is defined as follows:
```pascal
function SysSocketAccept(Socket: PSocket): PSocket;
var
  NextSocket, tmp: PSocket;
  Service: PNetworkService;
begin
  if not Socket.Blocking then
  begin
    Result := nil;
    Exit;
  end;
  Service := GetNetwork.SocketStream[Socket.SourcePort];
```
The Service variable points to a PNetworkService which is a socket listening on a port. The structure is defined as follows:
```pascal
  TNetworkService = record
    ServerSocket: PSocket;
    ClientSocket: PSocket;
  end;
```
The ServerSocket is the socket that is listening. The ClientSocket is a simple queue list in which incoming connections are enqueued.
```
  while True do
  begin
    NextSocket := Service.ClientSocket;
    while NextSocket <> nil do
    begin
      tmp := NextSocket;
      NextSocket := tmp.Next;
      if tmp.DispatcherEvent = DISP_ACCEPT then
      begin
        Dec(Service.ServerSocket.ConnectionsQueueCount);
        // TODO: This may be not needed because client socket are created based on the father socket
        tmp.Blocking := True;
        tmp.DispatcherEvent := DISP_ZOMBIE;
        Result := tmp;
        Exit;
      end;
    end;
    SysThreadSwitch;
  end;
end;
```
The function returns the head of the list of incoming connections. If the list is empty, it calls the scheduler. The function keeps calling the scheduler until an incoming connection arrives.

## SysSocketSelect()
* The next function is SysSocketSelect(). This function blocks until an event arrives to the socket, e.g., new data, timeout, connection closed. The function behaves differently depending whether the socket is blocking or not blocking. The non-blocking path is not implemented yet. In case the socket is blocking, the function first sets a timeout to wait for a new event.

```pascal
function SysSocketSelect(Socket: PSocket; TimeOut: LongInt): Boolean;
begin
  Result := True;
  if not Socket.Blocking then
  begin
    {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h TimeOut: %d\n', [PtrUInt(Socket), TimeOut]); {$ENDIF}
    if Socket.State = SCK_LOCALCLOSING then
    begin
      Socket.DispatcherEvent := DISP_CLOSE;
      {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h in SCK_LOCALCLOSING, executing DISP_CLOSE\n', [PtrUInt(Socket)]); {$ENDIF}
      Exit;
    end;
    if Socket.BufferReader < Socket.Buffer+Socket.BufferLength then
    begin
      Socket.DispatcherEvent := DISP_RECEIVE;
      {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h executing, DISP_RECEIVE\n', [PtrUInt(Socket)]); {$ENDIF}
      Exit;
    end;
    SetSocketTimeOut(Socket, TimeOut);
    {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h set timeout\n', [PtrUInt(Socket)]); {$ENDIF}
  end else
  begin
    SetSocketTimeOut(Socket, TimeOut);
    while True do
    begin
      {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h TimeOut: %d\n', [PtrUInt(Socket), TimeOut]); {$ENDIF}
      if Socket.State = SCK_LOCALCLOSING then
      begin
        Socket.DispatcherEvent := DISP_CLOSE;
        Result := False;
        {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h in SCK_LOCALCLOSING, executing DISP_CLOSE\n', [PtrUInt(Socket)]); {$ENDIF}
        Exit;
      end;
      if Socket.BufferReader < Socket.Buffer+Socket.BufferLength then
      begin
        Socket.DispatcherEvent := DISP_RECEIVE;
        {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h executing, DISP_RECEIVE\n', [PtrUInt(Socket)]); {$ENDIF}
        Exit;
      end;
      {$IFDEF DebugSocket} WriteDebug('SysSocketSelect: Socket %h set timeout\n', [PtrUInt(Socket)]); {$ENDIF}
      if Socket.TimeOut < read_rdtsc then
      begin
        Result := False;
        Exit;
      end;
      SysThreadSwitch;
    end;
  end;
end;
```
* The function checks for three conditions for exiting:
1. The remote peer has disconnected
2. There is new data in the incoming buffer
3. The timeout has running out of time
If none of these conditions is satisfied, the function calls the scheduler. Note that here the timeout is approximative, since the scheduler is cooperative, we do not know when the thread is going to be scheduler again. Other sort of timers are required for hard deadlines.


* The user can use the `UsedDefined` to store information for a given Socket. For example, this is used to store an extra buffer in the StaticWebServer example. For example, this is used in the StaticWebServer example:
```pascal
    HttpClient := SysSocketAccept(HttpServer);
    rq := ToroGetMem(sizeof(TRequest));
    rq.BufferStart := ToroGetMem(Max_Path_Len);
    rq.BufferEnd := rq.BufferStart;
    rq.counter := 0;
    HttpClient.UserDefined := rq;
    tid := BeginThread(nil, 4096*2, ProcessesSocket, HttpClient, 0, tid);
```

## SysSocketSend()
* Sending data through a socket happens with the `SysSocketSend()` function. This is a non-blocking function. The function only blocks if the receiving side is not able to get more data. In this case, the function calls the scheduler until the other side becomes available.
* The function requires one copy from the user's buffer.
* The function fills the vsock header.
* It sends the date by using SysNetWorkSend(). This is a non-blocking function. The packets are enqueued into the available ring of the virtio-vsock device.
* The function returns after sending all the packets
* Note that the packet is allocated in this function but it is not released. The release happens by the kernel when the driver informs that the packet has been sent or consumed by the device.

```pascal
function SysSocketSend(Socket: PSocket; Addr: PChar; AddrLen, Flags: UInt32): LongInt;
var
  Packet: PPacket;
  VPacket: PVirtIOVSocketPacket;
  P: PChar;
  FragLen: UInt32;
begin
  while Addrlen > 0 do
  begin
    if Addrlen > VIRTIO_VSOCK_MAX_PKT_BUF_SIZE then
      FragLen := VIRTIO_VSOCK_MAX_PKT_BUF_SIZE
    else
      FragLen := Addrlen;

    while FragLen = Socket.RemoteWinLen - Socket.RemoteWinCount do
    begin
      SysThreadSwitch;
    end;

    if FragLen > Socket.RemoteWinLen - Socket.RemoteWinCount then
      FragLen := Socket.RemoteWinLen - Socket.RemoteWinCount;

    Packet := AllocatePacket(sizeof(TVirtIOVSockHdr) + FragLen);
    VPacket := Pointer(Packet.Data);
    VPacket.hdr.src_cid := GetNetwork.NetworkInterface.Minor;
    VPacket.hdr.dst_cid := Socket.DestIp;
    VPacket.hdr.src_port := Socket.SourcePort;
    VPacket.hdr.dst_port := Socket.DestPort;
    VPacket.hdr.flags := 0;
    VPacket.hdr.op := VIRTIO_VSOCK_OP_RW;
    VPacket.hdr.tp := VIRTIO_VSOCK_TYPE_STREAM;
    VPacket.hdr.len := FragLen;
    VPacket.hdr.buf_alloc := MAX_WINDOW;
    VPacket.hdr.fwd_cnt := Socket.BufferLength;
    P := Pointer(@VPacket.data);
    Move(Addr^, P^, FragLen);
    SysNetworkSend(Packet);
    Dec(AddrLen, FragLen);
    Inc(Addr, FragLen);
  end;
    Result := AddrLen;
end;
```
## SysSocketRecv()
* This function reads a number of `AddrLen` bytes from the socket buffer and stores them into `Addr`. This function requires one-copy from the Socket buffer.
* The function immediately exits if:
1. The socket is not in  SCK_TRANSMITTING state
2. 0 bytes are requested to be read
3. There is no more data to be read.
The function returns the number of bytes read from the socket buffer.
The function consumes the data from the internal buffer.
* This function returns if there is no data in the internal buffer. This function is usually used together with Select() to block until data is received.
```pascal
function SysSocketRecv(Socket: PSocket; Addr: PChar; AddrLen, Flags: UInt32): LongInt;
var
  FragLen: LongInt;
  PendingBytes: LongInt;
begin
  {$IFDEF DebugSocket} WriteDebug('SysSocketRecv: BufferLength: %d\n', [Socket.BufferLength]); {$ENDIF}
  Result := 0;
  if (Socket.State <> SCK_TRANSMITTING) or (AddrLen=0) or (Socket.BufferReader = Socket.Buffer+Socket.BufferLength) then
  begin
    {$IFDEF DebugSocket} WriteDebug('SysSocketRecv -> Exit\n', []); {$ENDIF}
    Exit;
  end;
  while (AddrLen > 0) and (Socket.State = SCK_TRANSMITTING) do
  begin
    PendingBytes := Socket.BufferLength - (PtrUInt(Socket.BufferReader)-PtrUInt(Socket.Buffer));
    {$IFDEF DebugSocket} WriteDebug('SysSocketRecv: AddrLen: %d PendingBytes: %d\n', [AddrLen, PendingBytes]); {$ENDIF}
    if AddrLen > PendingBytes then
      FragLen := PendingBytes
    else
      FragLen := AddrLen;
    Dec(AddrLen, FragLen);
    Move(Socket.BufferReader^, Addr^, FragLen);
    {$IFDEF DebugSocket} WriteDebug('SysSocketRecv: Receiving from %h to %h count: %d\n', [PtrUInt(Socket.BufferReader), PtrUInt(Addr), FragLen]); {$ENDIF}
    Inc(Result, FragLen);
    Inc(Socket.BufferReader, FragLen);
    if Socket.BufferReader = Socket.Buffer+Socket.BufferLength then
    begin
      {$IFDEF DebugSocket} WriteDebug('SysSocketRecv: Resetting Socket.BufferReader\n', []); {$ENDIF}
      Socket.BufferReader := Socket.Buffer;
      Socket.BufferLength := 0;
      Break;
    end;
  end;
end;
```

## SysSocketPeek()
The function behaves similar than SysSocketRecv. The main difference is that the data is not consumed from the socket buffer. This is useful when we want to read a whole structure from the buffer. We keep calling SysSocketPeek() until it succeeds and then we get the whole structure by using SysSocketRecv().

```pascal
function SysSocketPeek(Socket: PSocket; Addr: PChar; AddrLen: UInt32): LongInt;
var
  FragLen: LongInt;
begin
  {$IFDEF DebugSocket} WriteDebug('SysSocketPeek BufferLength: %d\n', [Socket.BufferLength]); {$ENDIF}
  Result := 0;
  if (Socket.State <> SCK_TRANSMITTING) or (AddrLen=0) or (Socket.Buffer+Socket.BufferLength = Socket.BufferReader) then
  begin
    {$IFDEF DebugSocket} WriteDebug('SysSocketPeek -> Exit\n', []); {$ENDIF}
    Exit;
  end;
  while (AddrLen > 0) and (Socket.State = SCK_TRANSMITTING) do
  begin
    if Socket.BufferLength > AddrLen then
    begin
      FragLen := AddrLen;
      AddrLen := 0;
    end else
    begin
      FragLen := Socket.BufferLength;
      AddrLen := 0;
    end;
    Move(Socket.BufferReader^, Addr^, FragLen);
    {$IFDEF DebugSocket} WriteDebug('SysSocketPeek:  %q bytes from port %d to port %d\n', [PtrUInt(FragLen), Socket.SourcePort, Socket.DestPort]); {$ENDIF}
    Result := Result + FragLen;
  end;
end;
```
## SysSocketConnect()
This function connects a socket to a remote peer. It returns True if successes, or False otherwise. The function is defined as follows:
```pascal
function SysSocketConnect(Socket: PSocket): Boolean;
var
  Service: PNetworkService;
  Packet: PPacket;
  VPacket: PVirtIOVSocketPacket;
begin
  Result := False;
  Socket.Buffer := ToroGetMem(MAX_WINDOW);
  if Socket.Buffer = nil then
    Exit;
  Socket.SourcePort := GetFreePort;
  if Socket.SourcePort = 0 then
  begin
    ToroFreeMem(Socket.Buffer);
    Exit;
  end;
```
The function exits with error if there is not more memory for the local buffer or if there is not a local port that can be used as a source port.
```
  Socket.State := SCK_CONNECTING;
  Socket.mode := MODE_CLIENT;
  Socket.NeedFreePort := True;
  Socket.BufferLength :=0;
  Socket.BufferReader := Socket.Buffer;
  Service := ToroGetMem(sizeof(TNetworkService));
  if Service = nil then
  begin
    ToroFreeMem(Socket.Buffer);
    FreePort(Socket.SourcePort);
    Exit;
  end;
  GetNetwork.SocketStream[Socket.SourcePort]:= Service;
  Service.ServerSocket := Socket;
  Service.ClientSocket := Socket;
  Socket.Next := nil;
  Socket.SocketType := SOCKET_STREAM;
  Socket.BufferSender := nil;
  Socket.BufferSenderTail := nil;
  Packet := ToroGetMem(sizeof(TPacket)+ sizeof(TVirtIOVSockHdr));
  if Packet = nil then
  begin
    ToroFreeMem(Socket.Buffer);
    ToroFreeMem(GetNetwork.SocketStream[Socket.SourcePort]);
    GetNetwork.SocketStream[Socket.SourcePort] := nil;
    FreePort(Socket.SourcePort);
    Exit;
  end;
  Packet := AllocatePacket(sizeof(TVirtIOVSockHdr));
  VPacket := Pointer(Packet.Data);
  VPacket.hdr.src_cid := GetNetwork.NetworkInterface.Minor;
  VPacket.hdr.dst_cid := Socket.DestIp;
  VPacket.hdr.src_port := Socket.SourcePort;
  VPacket.hdr.dst_port := Socket.DestPort;
  VPacket.hdr.flags := 0;
  VPacket.hdr.op := VIRTIO_VSOCK_OP_REQUEST;
  VPacket.hdr.tp := VIRTIO_VSOCK_TYPE_STREAM;
  VPacket.hdr.len := 0;
  VPacket.hdr.buf_alloc := MAX_WINDOW;
  VPacket.hdr.fwd_cnt := Socket.BufferLength;
  SysNetworkSend(Packet);
  SetSocketTimeOut(Socket, WAIT_ACK);
```
This call builds an `OP_REQUEST` packet and sends it to the destination CID. It sets a timeout to wait for the remote response. If non response is received, the function returns with error.
```pascal
  while True do
  begin
    if (Socket.TimeOut < read_rdtsc) or (Socket.State = SCK_BLOCKED) then
    begin
      ToroFreeMem(Socket.Buffer);
      ToroFreeMem(GetNetwork.SocketStream[Socket.SourcePort]);
      GetNetwork.SocketStream[Socket.SourcePort] := nil;
      FreePort(Socket.SourcePort);
      Exit;
    end
    else if Socket.State = SCK_TRANSMITTING then
    begin
      Result := True;
      Exit;
    end;
    SysThreadSwitch;
  end;
end;
```
After the remote peer accepts the connection, the function exits with True and the socket in SCK_TRANSMITTING state.

## SysSocketClose()
* SysSocketClose():

```pascal
procedure SysSocketClose(Socket: PSocket);
var
  Packet: PPacket;
  VPacket: PVirtIOVSocketPacket;
begin
  if Socket.State = SCK_LOCALCLOSING then
  begin
    FreeSocket(Socket)
  end else
  begin
    Socket.State := SCK_PEER_DISCONNECTED;
    Packet := AllocatePacket(sizeof(TVirtIOVSockHdr));
    VPacket := Pointer(Packet.Data);
    VPacket.hdr.src_cid := GetNetwork.NetworkInterface.Minor;
    VPacket.hdr.dst_cid := Socket.DestIp;
    VPacket.hdr.src_port := Socket.SourcePort;
    VPacket.hdr.dst_port := Socket.DestPort;
    VPacket.hdr.flags := 3;
    VPacket.hdr.op := VIRTIO_VSOCK_OP_SHUTDOWN;
    VPacket.hdr.tp := VIRTIO_VSOCK_TYPE_STREAM;
    VPacket.hdr.len := 0;
    VPacket.hdr.buf_alloc := MAX_WINDOW;
    VPacket.hdr.fwd_cnt := Socket.BufferLength;
    SysNetworkSend(Packet);
  end;
end;
```
