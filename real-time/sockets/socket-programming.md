---
title: "Socket Programming and TCP in .NET"
date_created: 2025-04-02
date_updated: 2025-04-02
authors: ["Repository Maintainers"]
tags: ["networking", "tcp", "sockets", "communication", "low-level"]
difficulty: "intermediate"
---

# Socket Programming and TCP in .NET

## Overview

Socket programming provides direct access to network communication protocols, enabling developers to build custom networking solutions. This document focuses on TCP (Transmission Control Protocol) socket programming in .NET, covering both the lower-level Socket API and higher-level abstractions available in the framework. Understanding socket programming is essential for building high-performance networked applications, custom protocols, and when existing higher-level APIs don't meet specific requirements.

## Core Concepts

### What are Sockets?

Sockets are endpoints for communication between processes, either on the same machine or across a network. They provide a bidirectional communication channel with the following characteristics:

- Abstract interface to networking protocols
- Support for different communication patterns (stream, datagram)
- Address family independence (IPv4, IPv6)
- Connection-oriented (TCP) or connectionless (UDP) communication

### TCP/IP Fundamentals

TCP (Transmission Control Protocol) is a connection-oriented protocol that provides:

- Reliable data delivery
- Error detection and correction
- Flow control and congestion avoidance
- Ordered data transmission
- Stream-based communication

TCP operates on top of IP (Internet Protocol), which handles routing and addressing across networks.

### Socket Life Cycle

1. **Create**: Initialize a socket with specific address family, socket type, and protocol
2. **Bind**: Associate the socket with a local endpoint (IP and port)
3. **Listen/Connect**: Server listens for connections, client initiates connection
4. **Accept**: Server accepts incoming client connections
5. **Send/Receive**: Data exchange between endpoints
6. **Close**: Terminate the connection and release resources

### Socket API Architecture in .NET

.NET provides multiple layers of abstraction for socket programming:

1. **Low-level Socket API**: `System.Net.Sockets.Socket` class
2. **Mid-level APIs**: `TcpClient`, `TcpListener`, `NetworkStream`
3. **High-level APIs**: HttpClient, WCF, gRPC, SignalR

## Implementation

### Basic TCP Server with Socket API

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;

public class TcpSocketServer
{
    public void Start(string ipAddress, int port)
    {
        // Create a TCP/IP socket
        Socket listener = new Socket(AddressFamily.InterNetwork, 
            SocketType.Stream, ProtocolType.Tcp);
        
        try
        {
            // Bind the socket to the local endpoint
            IPEndPoint localEndPoint = new IPEndPoint(IPAddress.Parse(ipAddress), port);
            listener.Bind(localEndPoint);
            
            // Listen for incoming connections
            listener.Listen(100);  // Backlog size of 100
            
            Console.WriteLine($"Server started on {ipAddress}

## Advanced Scenarios

### Socket Options and Configuration

Socket options can fine-tune behavior and performance:

```csharp
public void ConfigureSocketOptions(Socket socket)
{
    // Disable Nagle's algorithm for lower latency
    socket.NoDelay = true;
    
    // Set send/receive buffer sizes
    socket.ReceiveBufferSize = 8192;
    socket.SendBufferSize = 8192;
    
    // Set socket timeouts
    socket.ReceiveTimeout = 30000; // 30 seconds
    socket.SendTimeout = 30000;    // 30 seconds
    
    // Keep-alive settings
    socket.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.KeepAlive, true);
    
    // For TCP fast recovery
    byte[] keepAliveValues = new byte[12];
    BitConverter.GetBytes(1).CopyTo(keepAliveValues, 0);                  // On/Off
    BitConverter.GetBytes(30000).CopyTo(keepAliveValues, 4);              // Time before first probe (ms)
    BitConverter.GetBytes(1000).CopyTo(keepAliveValues, 8);               // Interval between probes (ms)
    socket.IOControl(IOControlCode.KeepAliveValues, keepAliveValues, null);
    
    // Allow reuse of local address
    socket.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.ReuseAddress, true);
    
    // Set linger option for graceful close
    socket.LingerState = new LingerOption(true, 10); // Wait up to 10 seconds to send queued data
}
```

### Custom TCP Protocol Implementation

Building a custom protocol involves defining message formats and handling:

```csharp
public class CustomProtocol
{
    // Message format: [4-byte length][1-byte type][payload]
    
    public enum MessageType : byte
    {
        Request = 1,
        Response = 2,
        Event = 3,
        Error = 4
    }
    
    public class Message
    {
        public MessageType Type { get; set; }
        public byte[] Payload { get; set; }
    }
    
    public async Task SendMessageAsync(NetworkStream stream, MessageType type, byte[] payload)
    {
        // Calculate total message size
        int totalLength = 1 + payload.Length; // 1 byte for type + payload
        
        // Create the header (4-byte length)
        byte[] header = BitConverter.GetBytes(totalLength);
        
        // Create the message type byte
        byte[] typeBytes = new byte[] { (byte)type };
        
        // Send header
        await stream.WriteAsync(header, 0, header.Length);
        
        // Send type
        await stream.WriteAsync(typeBytes, 0, typeBytes.Length);
        
        // Send payload
        await stream.WriteAsync(payload, 0, payload.Length);
        await stream.FlushAsync();
    }
    
    public async Task<Message> ReceiveMessageAsync(NetworkStream stream)
    {
        // Read the length (4 bytes)
        byte[] lengthBuffer = new byte[4];
        await ReadExactlyAsync(stream, lengthBuffer, 4);
        int messageLength = BitConverter.ToInt32(lengthBuffer, 0);
        
        // Read message type (1 byte)
        byte[] typeBuffer = new byte[1];
        await ReadExactlyAsync(stream, typeBuffer, 1);
        MessageType type = (MessageType)typeBuffer[0];
        
        // Read payload
        int payloadLength = messageLength - 1; // Subtract type byte
        byte[] payload = new byte[payloadLength];
        await ReadExactlyAsync(stream, payload, payloadLength);
        
        return new Message
        {
            Type = type,
            Payload = payload
        };
    }
    
    private async Task ReadExactlyAsync(NetworkStream stream, byte[] buffer, int bytesToRead)
    {
        int totalBytesRead = 0;
        
        while (totalBytesRead < bytesToRead)
        {
            int bytesRead = await stream.ReadAsync(buffer, totalBytesRead, bytesToRead - totalBytesRead);
            
            if (bytesRead == 0)
                throw new EndOfStreamException("Connection closed prematurely");
                
            totalBytesRead += bytesRead;
        }
    
    public async Task SendAsync(string message, CancellationToken cancellationToken = default)
    {
        await _connectionLock.WaitAsync(cancellationToken);
        
        try
        {
            if (!IsConnected)
            {
                throw new InvalidOperationException("Client is not connected");
            }
            
            byte[] data = Encoding.UTF8.GetBytes(message + "\n");
            await _stream.WriteAsync(data, 0, data.Length, cancellationToken);
            await _stream.FlushAsync(cancellationToken);
        }
        finally
        {
            _connectionLock.Release();
        }
    }
    
    public async Task DisconnectAsync()
    {
        await _connectionLock.WaitAsync();
        
        try
        {
            CleanupConnection();
        }
        finally
        {
            _connectionLock.Release();
        }
    }
    
    private async Task ReceiveMessagesAsync(CancellationToken cancellationToken)
    {
        byte[] buffer = new byte[4096];
        StringBuilder messageBuilder = new StringBuilder();
        
        try
        {
            while (!cancellationToken.IsCancellationRequested && IsConnected)
            {
                int bytesRead = await _stream.ReadAsync(buffer, 0, buffer.Length, cancellationToken);
                
                if (bytesRead == 0)
                {
                    // Connection closed by the server
                    break;
                }
                
                string data = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                messageBuilder.Append(data);
                
                // Process complete messages (ending with newline)
                string accumulated = messageBuilder.ToString();
                int newlineIndex;
                
                while ((newlineIndex = accumulated.IndexOf('\n')) >= 0)
                {
                    string message = accumulated.Substring(0, newlineIndex).Trim();
                    accumulated = accumulated.Substring(newlineIndex + 1);
                    
                    OnMessageReceived(message);
                }
                
                // Update the StringBuilder with any remaining incomplete message
                messageBuilder.Clear();
                messageBuilder.Append(accumulated);
            }
        }
        catch (OperationCanceledException)
        {
            // Expected when cancellation is requested
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error receiving messages: {ex.Message}");
        }
        finally
        {
            OnDisconnected();
            
            // Try to reconnect automatically
            _ = Task.Run(async () => 
            {
                try
                {
                    if (!_isDisposed && !_cts.IsCancellationRequested)
                    {
                        await ReconnectAsync(_cts.Token);
                    }
                }
                catch
                {
                    // Ignore reconnection errors
                }
            });
        }
    }
    
    private void CleanupConnection()
    {
        _stream?.Dispose();
        _stream = null;
        
        if (_client != null)
        {
            try
            {
                _client.Close();
            }
            catch
            {
                // Ignore errors during cleanup
            }
            
            _client = null;
            OnDisconnected();
        }
    }
    
    protected virtual void OnMessageReceived(string message)
    {
        MessageReceived?.Invoke(this, message);
    }
    
    protected virtual void OnConnected()
    {
        Connected?.Invoke(this, EventArgs.Empty);
    }
    
    protected virtual void OnDisconnected()
    {
        Disconnected?.Invoke(this, EventArgs.Empty);
    }
    
    public void Dispose()
    {
        if (!_isDisposed)
        {
            _isDisposed = true;
            
            _cts.Cancel();
            _cts.Dispose();
            
            CleanupConnection();
            _connectionLock.Dispose();
        }
    }
}

// Usage example
public static class ClientExample
{
    public static async Task RunClientExampleAsync()
    {
        using var client = new ReliableTcpClient("127.0.0.1", 8080);
        
        client.MessageReceived += (sender, message) =>
        {
            Console.WriteLine($"Received: {message}");
        };
        
        client.Connected += (sender, e) =>
        {
            Console.WriteLine("Connected event received");
        };
        
        client.Disconnected += (sender, e) =>
        {
            Console.WriteLine("Disconnected event received");
        };
        
        try
        {
            await client.ConnectAsync();
            
            // Send some example messages
            await client.SendAsync("Hello, server!");
            await Task.Delay(1000);
            await client.SendAsync("How are you today?");
            
            // Keep the application running
            Console.WriteLine("Press Enter to exit");
            Console.ReadLine();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error: {ex.Message}");
        }
    }
}
```

### TCP Server with Connection Limits and Monitoring

```csharp
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

public class MonitoredTcpServer : IDisposable
{
    private readonly TcpListener _listener;
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    private readonly ConcurrentDictionary<string, ClientConnection> _activeConnections = 
        new ConcurrentDictionary<string, ClientConnection>();
    private readonly SemaphoreSlim _connectionLimiter;
    private readonly Timer _monitoringTimer;
    private readonly ServerStatistics _statistics = new ServerStatistics();
    
    public int MaxConnections { get; }
    public ServerStatistics Statistics => _statistics;
    
    public MonitoredTcpServer(string ipAddress, int port, int maxConnections)
    {
        _listener = new TcpListener(IPAddress.Parse(ipAddress), port);
        MaxConnections = maxConnections;
        _connectionLimiter = new SemaphoreSlim(maxConnections, maxConnections);
        
        // Create timer for monitoring server statistics (every 5 seconds)
        _monitoringTimer = new Timer(MonitorServerStatus, null, TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(5));
    }
    
    public async Task StartAsync()
    {
        _listener.Start();
        Console.WriteLine($"Server started on {((IPEndPoint)_listener.LocalEndpoint)}");
        Console.WriteLine($"Max connections: {MaxConnections}");
        
        try
        {
            while (!_cts.IsCancellationRequested)
            {
                // Wait for an available connection slot
                await _connectionLimiter.WaitAsync(_cts.Token);
                
                try
                {
                    // Accept the next client connection
                    TcpClient client = await _listener.AcceptTcpClientAsync();
                    string connectionId = Guid.NewGuid().ToString();
                    
                    // Create and track the client connection
                    var clientConnection = new ClientConnection(connectionId, client);
                    _activeConnections[connectionId] = clientConnection;
                    
                    // Update statistics
                    _statistics.IncrementTotalConnections();
                    _statistics.IncrementActiveConnections();
                    
                    // Start processing the client asynchronously
                    _ = ProcessClientAsync(clientConnection);
                }
                catch (OperationCanceledException)
                {
                    // Server is shutting down
                    break;
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Error accepting client: {ex.Message}");
                    _connectionLimiter.Release();
                }
            }
        }
        finally
        {
            _listener.Stop();
            Console.WriteLine("Server stopped");
        }
    }
    
    private async Task ProcessClientAsync(ClientConnection client)
    {
        string clientEndPoint = ((IPEndPoint)client.Client.Client.RemoteEndPoint).ToString();
        Console.WriteLine($"Client connected: {clientEndPoint} (ID: {client.ConnectionId})");
        
        try
        {
            Stopwatch connectionTimer = Stopwatch.StartNew();
            NetworkStream stream = client.Client.GetStream();
            byte[] buffer = new byte[8192];
            
            while (!_cts.IsCancellationRequested && client.Client.Connected)
            {
                // Set a read timeout
                var readTask = stream.ReadAsync(buffer, 0, buffer.Length, _cts.Token);
                int bytesRead;
                
                try
                {
                    bytesRead = await readTask.WaitAsync(TimeSpan.FromSeconds(30), _cts.Token);
                }
                catch (TimeoutException)
                {
                    Console.WriteLine($"Client {clientEndPoint} timed out");
                    break;
                }
                
                if (bytesRead == 0)
                {
                    // Client disconnected properly
                    break;
                }
                
                // Update statistics
                _statistics.AddBytesReceived(bytesRead);
                
                // Process the message
                string message = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                Console.WriteLine($"Received from {clientEndPoint}: {message}");
                
                // Create response
                string response = $"Echo: {message}";
                byte[] responseData = Encoding.UTF8.GetBytes(response);
                
                // Send the response
                await stream.WriteAsync(responseData, 0, responseData.Length);
                
                // Update statistics
                _statistics.AddBytesSent(responseData.Length);
            }
            
            // Update connection time statistics
            connectionTimer.Stop();
            _statistics.AddConnectionTime(connectionTimer.Elapsed);
        }
        catch (OperationCanceledException)
        {
            // Server is shutting down
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error processing client {clientEndPoint}: {ex.Message}");
            _statistics.IncrementErrors();
        }
        finally
        {
            // Clean up the connection
            if (_activeConnections.TryRemove(client.ConnectionId, out _))
            {
                try
                {
                    client.Client.Close();
                    _statistics.DecrementActiveConnections();
                    Console.WriteLine($"Client disconnected: {clientEndPoint} (ID: {client.ConnectionId})");
                }
                catch
                {
                    // Ignore errors during cleanup
                }
            }
            
            // Release connection slot
            _connectionLimiter.Release();
        }
    }
    
    private void MonitorServerStatus(object state)
    {
        Console.WriteLine($"\n--- SERVER STATUS AT {DateTime.Now:HH:mm:ss} ---");
        Console.WriteLine($"Active connections: {_statistics.ActiveConnections}");
        Console.WriteLine($"Total connections: {_statistics.TotalConnections}");
        Console.WriteLine($"Bytes received: {FormatBytes(_statistics.BytesReceived)}");
        Console.WriteLine($"Bytes sent: {FormatBytes(_statistics.BytesSent)}");
        Console.WriteLine($"Avg connection time: {_statistics.AverageConnectionTime:g}");
        Console.WriteLine($"Errors: {_statistics.ErrorCount}");
        Console.WriteLine("------------------------------------\n");
    }
    
    private string FormatBytes(long bytes)
    {
        string[] suffixes = { "B", "KB", "MB", "GB" };
        int suffixIndex = 0;
        double size = bytes;
        
        while (size >= 1024 && suffixIndex < suffixes.Length - 1)
        {
            suffixIndex++;
            size /= 1024;
        }
        
        return $"{size:F2} {suffixes[suffixIndex]}";
    }
    
    public async Task StopAsync()
    {
        if (!_cts.IsCancellationRequested)
        {
            // Signal cancellation
            _cts.Cancel();
            
            // Close all client connections
            foreach (var connection in _activeConnections.Values)
            {
                try
                {
                    connection.Client.Close();
                }
                catch
                {
                    // Ignore errors during cleanup
                }
            }
            
            // Clear active connections
            _activeConnections.Clear();
        }
        
        await Task.CompletedTask;
    }
    
    public void Dispose()
    {
        _cts.Cancel();
        _cts.Dispose();
        _connectionLimiter.Dispose();
        _monitoringTimer.Dispose();
        
        // Make sure to stop the listener
        try
        {
            _listener.Stop();
        }
        catch
        {
            // Ignore errors during cleanup
        }
    }
    
    private class ClientConnection
    {
        public string ConnectionId { get; }
        public TcpClient Client { get; }
        
        public ClientConnection(string connectionId, TcpClient client)
        {
            ConnectionId = connectionId;
            Client = client;
        }
    }
    
    public class ServerStatistics
    {
        private int _totalConnections;
        private int _activeConnections;
        private long _bytesReceived;
        private long _bytesSent;
        private long _totalConnectionTimeMs;
        private int _closedConnections;
        private int _errors;
        
        public int TotalConnections => _totalConnections;
        public int ActiveConnections => _activeConnections;
        public long BytesReceived => _bytesReceived;
        public long BytesSent => _bytesSent;
        public TimeSpan AverageConnectionTime => _closedConnections > 0 
            ? TimeSpan.FromMilliseconds(_totalConnectionTimeMs / _closedConnections) 
            : TimeSpan.Zero;
        public int ErrorCount => _errors;
        
        public void IncrementTotalConnections()
        {
            Interlocked.Increment(ref _totalConnections);
        }
        
        public void IncrementActiveConnections()
        {
            Interlocked.Increment(ref _activeConnections);
        }
        
        public void DecrementActiveConnections()
        {
            Interlocked.Decrement(ref _activeConnections);
            Interlocked.Increment(ref _closedConnections);
        }
        
        public void AddBytesReceived(long bytes)
        {
            Interlocked.Add(ref _bytesReceived, bytes);
        }
        
        public void AddBytesSent(long bytes)
        {
            Interlocked.Add(ref _bytesSent, bytes);
        }
        
        public void AddConnectionTime(TimeSpan connectionTime)
        {
            Interlocked.Add(ref _totalConnectionTimeMs, (long)connectionTime.TotalMilliseconds);
        }
        
        public void IncrementErrors()
        {
            Interlocked.Increment(ref _errors);
        }
    }
}
```

## Further Reading

- [.NET Socket Programming Guide](https://docs.microsoft.com/en-us/dotnet/framework/network-programming/socket-programming)
- [System.Net.Sockets Namespace](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets)
- [TCP/IP Illustrated, Volume 1: The Protocols](https://www.amazon.com/TCP-Illustrated-Protocols-Addison-Wesley-Professional/dp/0321336313) by Kevin R. Fall and W. Richard Stevens
- [Network Programming in .NET with C# and Visual Basic](https://www.amazon.com/Network-Programming-Visual-Basic-NET/dp/1590590759) by Fiach Reid
- [High-Performance Browser Networking](https://hpbn.co/) by Ilya Grigorik (for understanding network protocols)

## Related Topics

- [Real-time Communication with SignalR](../../real-time/signalr/introduction.md)
- [WebSocket Protocol](../../real-time/websockets/websocket-protocol.md)
- [gRPC Services](../../api-development/grpc/grpc-services.md)
- [ASP.NET Core Web API](../../api-development/rest/rest-api-development.md)
- [Network Optimization Techniques](../../performance/optimization/network-optimization.md)
```

### Socket-Based IPC (Inter-Process Communication)

Sockets can provide efficient IPC between processes on the same machine:

```csharp
public class NamedPipeServer
{
    public async Task StartAsync(string pipeName, CancellationToken cancellationToken = default)
    {
        // Create a Unix Domain Socket server (for Windows 10 1803+ and Linux/macOS)
        var endpoint = new UnixDomainSocketEndPoint($@"\\.\\pipe\\{pipeName}");
        
        using var server = new Socket(AddressFamily.Unix, SocketType.Stream, ProtocolType.Unspecified);
        
        try
        {
            server.Bind(endpoint);
            server.Listen(10);
            
            Console.WriteLine($"IPC server started on pipe: {pipeName}");
            
            while (!cancellationToken.IsCancellationRequested)
            {
                Socket client = await server.AcceptAsync(cancellationToken);
                _ = Task.Run(() => HandleClientAsync(client, cancellationToken));
            }
        }
        catch (OperationCanceledException)
        {
            // Normal cancellation
        }
        catch (Exception ex)
        {
            Console.WriteLine($"IPC server error: {ex.Message}");
        }
        finally
        {
            server.Close();
        }
    }
    
    private async Task HandleClientAsync(Socket client, CancellationToken cancellationToken)
    {
        using (client)
        {
            var buffer = new byte[4096];
            
            try
            {
                while (!cancellationToken.IsCancellationRequested)
                {
                    int bytesRead = await client.ReceiveAsync(buffer, SocketFlags.None, cancellationToken);
                    
                    if (bytesRead == 0)
                        break;
                        
                    string message = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                    Console.WriteLine($"IPC received: {message}");
                    
                    // Process message and send response
                    string response = $"Processed: {message}";
                    byte[] responseBytes = Encoding.UTF8.GetBytes(response);
                    await client.SendAsync(responseBytes, SocketFlags.None, cancellationToken);
                }
            }
            catch (OperationCanceledException)
            {
                // Normal cancellation
            }
            catch (Exception ex)
            {
                Console.WriteLine($"IPC client error: {ex.Message}");
            }
        }
    }
}

public class NamedPipeClient
{
    private Socket _client;
    
    public async Task ConnectAsync(string pipeName, CancellationToken cancellationToken = default)
    {
        var endpoint = new UnixDomainSocketEndPoint($@"\\.\\pipe\\{pipeName}");
        _client = new Socket(AddressFamily.Unix, SocketType.Stream, ProtocolType.Unspecified);
        
        await _client.ConnectAsync(endpoint, cancellationToken);
        Console.WriteLine($"Connected to IPC server on pipe: {pipeName}");
    }
    
    public async Task<string> SendMessageAsync(string message, CancellationToken cancellationToken = default)
    {
        byte[] messageBytes = Encoding.UTF8.GetBytes(message);
        await _client.SendAsync(messageBytes, SocketFlags.None, cancellationToken);
        
        var buffer = new byte[4096];
        int bytesRead = await _client.ReceiveAsync(buffer, SocketFlags.None, cancellationToken);
        
        return Encoding.UTF8.GetString(buffer, 0, bytesRead);
    }
    
    public void Disconnect()
    {
        _client?.Close();
        _client = null;
    }
}
```

### Multicast UDP Sockets

For scenarios requiring data distribution to multiple receivers:

```csharp
public class MulticastExample
{
    public static async Task RunMulticastSenderAsync(string multicastGroup, int port, string message)
    {
        using var client = new UdpClient();
        var endpoint = new IPEndPoint(IPAddress.Parse(multicastGroup), port);
        
        try
        {
            byte[] data = Encoding.UTF8.GetBytes(message);
            int sent = await client.SendAsync(data, data.Length, endpoint);
            Console.WriteLine($"Sent {sent} bytes to multicast group {multicastGroup}:{port}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Multicast send error: {ex.Message}");
        }
    }
    
    public static async Task RunMulticastReceiverAsync(
        string multicastGroup, 
        int port, 
        CancellationToken cancellationToken = default)
    {
        using var client = new UdpClient();
        
        // Configure UDP client for multicast
        client.ExclusiveAddressUse = false;
        client.Client.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.ReuseAddress, true);
        client.Client.Bind(new IPEndPoint(IPAddress.Any, port));
        
        // Join the multicast group
        client.JoinMulticastGroup(IPAddress.Parse(multicastGroup));
        
        Console.WriteLine($"Listening for multicast on {multicastGroup}:{port}");
        
        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                var result = await client.ReceiveAsync(cancellationToken);
                string message = Encoding.UTF8.GetString(result.Buffer);
                Console.WriteLine($"Received: {message} from {result.RemoteEndPoint}");
            }
        }
        catch (OperationCanceledException)
        {
            // Normal cancellation
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Multicast receive error: {ex.Message}");
        }
        finally
        {
            // Leave the multicast group
            client.DropMulticastGroup(IPAddress.Parse(multicastGroup));
        }
    }
}
```

### High-Performance Socket Server with Pipelines

Using System.IO.Pipelines for efficient data processing:

```csharp
public class PipelineSocketServer
{
    private readonly Socket _listener;
    
    public PipelineSocketServer(string ipAddress, int port)
    {
        _listener = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        _listener.Bind(new IPEndPoint(IPAddress.Parse(ipAddress), port));
    }
    
    public async Task RunAsync(CancellationToken cancellationToken = default)
    {
        _listener.Listen(100);
        Console.WriteLine("Server started and listening for connections");
        
        while (!cancellationToken.IsCancellationRequested)
        {
            var client = await _listener.AcceptAsync(cancellationToken);
            _ = ProcessClientAsync(client, cancellationToken);
        }
    }
    
    private async Task ProcessClientAsync(Socket client, CancellationToken cancellationToken)
    {
        Console.WriteLine($"Client connected: {client.RemoteEndPoint}");
        
        try
        {
            // Create a pipe for the socket
            var pipe = new Pipe();
            Task writing = FillPipeAsync(client, pipe.Writer, cancellationToken);
            Task reading = ReadPipeAsync(client, pipe.Reader, cancellationToken);
            
            // Wait for either task to complete
            await Task.WhenAny(reading, writing);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error processing client: {ex.Message}");
        }
        finally
        {
            Console.WriteLine($"Client disconnected: {client.RemoteEndPoint}");
            client.Close();
        }
    }
    
    private async Task FillPipeAsync(Socket socket, PipeWriter writer, CancellationToken cancellationToken)
    {
        const int minimumBufferSize = 4096;
        
        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                // Get memory from the PipeWriter
                Memory<byte> memory = writer.GetMemory(minimumBufferSize);
                
                // Read data from the Socket into the Memory
                int bytesRead = await socket.ReceiveAsync(memory, SocketFlags.None, cancellationToken);
                
                if (bytesRead == 0)
                {
                    break; // Client disconnected
                }
                
                // Tell the PipeWriter how much was read
                writer.Advance(bytesRead);
                
                // Make the data available to the PipeReader
                FlushResult result = await writer.FlushAsync(cancellationToken);
                
                if (result.IsCompleted)
                {
                    break;
                }
            }
        }
        catch (OperationCanceledException)
        {
            // Normal cancellation
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error in pipe writer: {ex.Message}");
        }
        finally
        {
            await writer.CompleteAsync();
        }
    }
    
    private async Task ReadPipeAsync(Socket socket, PipeReader reader, CancellationToken cancellationToken)
    {
        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                ReadResult result = await reader.ReadAsync(cancellationToken);
                ReadOnlySequence<byte> buffer = result.Buffer;
                
                // Process the data
                await ProcessDataAsync(socket, buffer, cancellationToken);
                
                // Mark the data as processed
                reader.AdvanceTo(buffer.End);
                
                if (result.IsCompleted)
                {
                    break;
                }
            }
        }
        catch (OperationCanceledException)
        {
            // Normal cancellation
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error in pipe reader: {ex.Message}");
        }
        finally
        {
            await reader.CompleteAsync();
        }
    }
    
    private async Task ProcessDataAsync(Socket socket, ReadOnlySequence<byte> buffer, CancellationToken cancellationToken)
    {
        // Example: Echo the data back to the client
        foreach (ReadOnlyMemory<byte> segment in buffer)
        {
            await socket.SendAsync(segment, SocketFlags.None, cancellationToken);
        }
    }
}
```

## Performance Considerations

### Socket Performance Metrics

Key metrics to monitor for socket performance:

- **Throughput**: Amount of data transferred per unit time
- **Latency**: Time for a request to be processed and returned
- **Connection Rate**: Number of new connections per second
- **Connection Time**: Time to establish a connection
- **Error Rate**: Failed connections or transmission errors
- **Resource Usage**: CPU, memory, network bandwidth consumption

### Improving Socket Performance

Techniques for optimizing socket performance:

1. **Minimize Allocations**:
   ```csharp
   // Pre-allocate buffers
   private readonly byte[] _buffer = new byte[8192];
   
   // Use ArrayPool for temporary buffers
   public async Task ProcessDataAsync(int size)
   {
       byte[] buffer = ArrayPool<byte>.Shared.Rent(size);
       try
       {
           await socket.ReceiveAsync(buffer, SocketFlags.None);
           // Process data...
       }
       finally
       {
           ArrayPool<byte>.Shared.Return(buffer);
       }
   }
   ```

2. **Batching**:
   ```csharp
   // Send multiple small messages in a single packet
   public async Task SendBatchAsync(List<string> messages)
   {
       using var ms = new MemoryStream();
       using var writer = new BinaryWriter(ms);
       
       // Write message count
       writer.Write(messages.Count);
       
       // Write all messages
       foreach (var message in messages)
       {
           byte[] data = Encoding.UTF8.GetBytes(message);
           writer.Write(data.Length);
           writer.Write(data);
       }
       
       // Send the entire batch
       byte[] batchData = ms.ToArray();
       await socket.SendAsync(batchData, SocketFlags.None);
   }
   ```

3. **Compression**:
   ```csharp
   public byte[] CompressData(byte[] data)
   {
       using var compressedStream = new MemoryStream();
       using (var gzipStream = new GZipStream(compressedStream, CompressionLevel.Fastest))
       {
           gzipStream.Write(data, 0, data.Length);
       }
       
       return compressedStream.ToArray();
   }
   ```

4. **Socket Pooling**:
   ```csharp
   public class SocketPool
   {
       private readonly ConcurrentBag<Socket> _pool = new ConcurrentBag<Socket>();
       private readonly string _host;
       private readonly int _port;
       private int _count;
       private readonly int _maxPoolSize;
       private readonly SemaphoreSlim _semaphore;
       
       public SocketPool(string host, int port, int maxPoolSize)
       {
           _host = host;
           _port = port;
           _maxPoolSize = maxPoolSize;
           _semaphore = new SemaphoreSlim(maxPoolSize, maxPoolSize);
       }
       
       public async Task<Socket> RentAsync(CancellationToken cancellationToken = default)
       {
           await _semaphore.WaitAsync(cancellationToken);
           
           if (_pool.TryTake(out Socket socket) && socket.Connected)
           {
               return socket;
           }
           
           // Create new socket
           var newSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
           await newSocket.ConnectAsync(_host, _port, cancellationToken);
           return newSocket;
       }
       
       public void Return(Socket socket)
       {
           if (socket == null || !socket.Connected)
           {
               socket?.Dispose();
               _semaphore.Release();
               return;
           }
           
           _pool.Add(socket);
           _semaphore.Release();
       }
       
       public void Dispose()
       {
           foreach (var socket in _pool)
           {
               try
               {
                   socket.Close();
                   socket.Dispose();
               }
               catch
               {
                   // Ignore errors during cleanup
               }
           }
           
           _pool.Clear();
           _semaphore.Dispose();
       }
   }
   ```

## Security Implications

### Common Socket Security Vulnerabilities

1. **Unencrypted Communications**: Sensitive data transmitted in plaintext
2. **Denial of Service (DoS)**: Resource exhaustion from excessive connections
3. **Man-in-the-Middle Attacks**: Interception of network traffic
4. **Buffer Overflows**: Input validation issues leading to memory corruption
5. **Connection Hijacking**: Unauthorized takeover of established connections

### Implementing Secure Socket Communication

```csharp
public class SecureSocketExample
{
    public static async Task<NetworkStream> CreateSecureServerStreamAsync(
        TcpClient client, 
        X509Certificate serverCertificate)
    {
        // Create an SSL stream
        SslStream sslStream = new SslStream(
            client.GetStream(), 
            false, 
            null, 
            null, 
            EncryptionPolicy.RequireEncryption);
        
        // Server authentication
        await sslStream.AuthenticateAsServerAsync(
            serverCertificate,
            clientCertificateRequired: false,
            SslProtocols.Tls13,
            checkCertificateRevocation: true);
            
        return sslStream;
    }
    
    public static void ConfigureServerSecurity(TcpListener listener)
    {
        // Set socket options for security
        listener.Server.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.DontLinger, true);
        
        // Set connection timeout
        listener.Server.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.ReceiveTimeout, 30000);
        listener.Server.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.SendTimeout, 30000);
        
        // Limit pending connections queue
        listener.Start(10);
    }
    
    public static void ValidateInput(byte[] buffer, int bytesReceived)
    {
        // Check for valid size
        if (bytesReceived > 8192)
        {
            throw new SecurityException("Message size exceeds maximum allowed");
        }
        
        // Check for valid content (example validation)
        bool isValid = true;
        for (int i = 0; i < bytesReceived; i++)
        {
            if (buffer[i] < 32 && buffer[i] != 10 && buffer[i] != 13)
            {
                isValid = false;
                break;
            }
        }
        
        if (!isValid)
        {
            throw new SecurityException("Message contains invalid characters");
        }
    }
    
    public static void ImplementRateLimiting(string clientIp, Dictionary<string, int> connectionCounts)
    {
        const int maxConnectionsPerClient = 5;
        
        lock (connectionCounts)
        {
            if (!connectionCounts.TryGetValue(clientIp, out int count))
            {
                connectionCounts[clientIp] = 1;
            }
            else
            {
                if (count >= maxConnectionsPerClient)
                {
                    throw new SecurityException($"Connection limit exceeded for {clientIp}");
                }
                
                connectionCounts[clientIp] = count + 1;
            }
        }
    }
}
```

## Code Examples

### Complete TCP Chat Server

```csharp
using System;
using System.Collections.Concurrent;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

public class ChatServer
{
    private readonly TcpListener _listener;
    private readonly ConcurrentDictionary<string, ChatClient> _clients = new ConcurrentDictionary<string, ChatClient>();
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    
    public ChatServer(string ipAddress, int port)
    {
        _listener = new TcpListener(IPAddress.Parse(ipAddress), port);
    }
    
    public async Task StartAsync()
    {
        _listener.Start();
        Console.WriteLine($"Chat server started on {((IPEndPoint)_listener.LocalEndpoint).Port}");
        
        try
        {
            while (!_cts.Token.IsCancellationRequested)
            {
                TcpClient client = await _listener.AcceptTcpClientAsync();
                _ = HandleClientAsync(client);
            }
        }
        catch (OperationCanceledException)
        {
            // Shutdown requested
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Server error: {ex.Message}");
        }
        finally
        {
            await StopAsync();
        }
    }
    
    public async Task StopAsync()
    {
        if (!_cts.IsCancellationRequested)
        {
            _cts.Cancel();
            
            // Disconnect all clients
            foreach (var client in _clients.Values)
            {
                await client.DisconnectAsync();
            }
            
            _clients.Clear();
            _listener.Stop();
            Console.WriteLine("Chat server stopped");
        }
    }
    
    private async Task HandleClientAsync(TcpClient client)
    {
        string clientId = Guid.NewGuid().ToString();
        var chatClient = new ChatClient(clientId, client);
        string clientEndpoint = ((IPEndPoint)client.Client.RemoteEndPoint).ToString();
        
        try
        {
            // Request a username
            await chatClient.SendAsync("Welcome! Please enter your username:");
            string username = await chatClient.ReceiveAsync();
            
            if (string.IsNullOrWhiteSpace(username))
            {
                await chatClient.SendAsync("Invalid username. Disconnecting.");
                chatClient.Disconnect();
                return;
            }
            
            chatClient.Username = username;
            _clients.TryAdd(clientId, chatClient);
            
            // Notify all clients of the new user
            await BroadcastAsync($"{username} has joined the chat.", clientId);
            Console.WriteLine($"{username} ({clientEndpoint}) connected");
            
            // Start receiving messages
            while (!_cts.Token.IsCancellationRequested && client.Connected)
            {
                string message = await chatClient.ReceiveAsync();
                
                if (string.IsNullOrEmpty(message))
                {
                    break; // Client disconnected
                }
                
                Console.WriteLine($"{username}: {message}");
                await BroadcastAsync($"{username}: {message}", clientId);
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error handling client {clientEndpoint}: {ex.Message}");
        }
        finally
        {
            // Remove and disconnect the client
            if (_clients.TryRemove(clientId, out ChatClient removedClient))
            {
                Console.WriteLine($"{removedClient.Username} ({clientEndpoint}) disconnected");
                
                await BroadcastAsync($"{removedClient.Username} has left the chat.", clientId);
                removedClient.Disconnect();
            }
        }
    }
    
    private async Task BroadcastAsync(string message, string excludeClientId = null)
    {
        var tasks = new List<Task>();
        
        foreach (var client in _clients.Values)
        {
            if (client.ClientId != excludeClientId)
            {
                tasks.Add(client.SendAsync(message));
            }
        }
        
        await Task.WhenAll(tasks);
    }
    
    private class ChatClient
    {
        private readonly TcpClient _client;
        private readonly NetworkStream _stream;
        private readonly StreamReader _reader;
        private readonly StreamWriter _writer;
        
        public string ClientId { get; }
        public string Username { get; set; }
        
        public ChatClient(string clientId, TcpClient client)
        {
            ClientId = clientId;
            _client = client;
            _stream = client.GetStream();
            _reader = new StreamReader(_stream);
            _writer = new StreamWriter(_stream) { AutoFlush = true };
        }
        
        public async Task SendAsync(string message)
        {
            try
            {
                await _writer.WriteLineAsync(message);
            }
            catch
            {
                // Ignore errors when sending
            }
        }
        
        public async Task<string> ReceiveAsync()
        {
            try
            {
                return await _reader.ReadLineAsync();
            }
            catch
            {
                return null; // Connection likely closed
            }
        }
        
        public void Disconnect()
        {
            _reader?.Dispose();
            _writer?.Dispose();
            _stream?.Dispose();
            _client?.Close();
        }
        
        public async Task DisconnectAsync()
        {
            try
            {
                await SendAsync("Server is shutting down. Goodbye!");
            }
            finally
            {
                Disconnect();
            }
        }
    }
}

// Usage
public static class Program
{
    public static async Task Main(string[] args)
    {
        var server = new ChatServer("127.0.0.1", 8080);
        
        // Handle shutdown gracefully
        Console.CancelKeyPress += async (sender, e) =>
        {
            e.Cancel = true;
            await server.StopAsync();
        };
        
        await server.StartAsync();
    }
}
```

### Async TCP Client with Reconnection Logic

```csharp
using System;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

public class ReliableTcpClient : IDisposable
{
    private readonly string _hostname;
    private readonly int _port;
    private TcpClient _client;
    private NetworkStream _stream;
    private CancellationTokenSource _cts;
    private Task _receiveTask;
    private readonly SemaphoreSlim _connectionLock = new SemaphoreSlim(1, 1);
    private readonly TimeSpan _reconnectDelay = TimeSpan.FromSeconds(5);
    private bool _isDisposed;
    
    public event EventHandler<string> MessageReceived;
    public event EventHandler Connected;
    public event EventHandler Disconnected;
    
    public bool IsConnected => _client?.Connected ?? false;
    
    public ReliableTcpClient(string hostname, int port)
    {
        _hostname = hostname;
        _port = port;
        _cts = new CancellationTokenSource();
    }
    
    public async Task ConnectAsync(CancellationToken cancellationToken = default)
    {
        await _connectionLock.WaitAsync(cancellationToken);
        
        try
        {
            if (IsConnected)
                return;
                
            _client = new TcpClient();
            
            try
            {
                await _client.ConnectAsync(_hostname, _port);
                _stream = _client.GetStream();
                
                // Start receiving messages
                var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(_cts.Token, cancellationToken);
                _receiveTask = ReceiveMessagesAsync(linkedCts.Token);
                
                OnConnected();
                Console.WriteLine($"Connected to {_hostname}:{_port}");
            }
            catch (Exception ex)
            {
                CleanupConnection();
                throw new Exception($"Failed to connect to {_hostname}:{_port}", ex);
            }
        }
        finally
        {
            _connectionLock.Release();
        }
    }
    
    public async Task<bool> ReconnectAsync(CancellationToken cancellationToken = default)
    {
        await _connectionLock.WaitAsync(cancellationToken);
        
        try
        {
            // Clean up existing connection
            CleanupConnection();
            
            // Attempt to reconnect
            bool reconnected = false;
            int attempts = 0;
            
            while (!reconnected && !cancellationToken.IsCancellationRequested && attempts < 5)
            {
                attempts++;
                
                try
                {
                    Console.WriteLine($"Reconnection attempt {attempts}...");
                    await Task.Delay(_reconnectDelay, cancellationToken);
                    
                    _client = new TcpClient();
                    await _client.ConnectAsync(_hostname, _port);
                    _stream = _client.GetStream();
                    
                    // Start receiving messages again
                    var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(_cts.Token, cancellationToken);
                    _receiveTask = ReceiveMessagesAsync(linkedCts.Token);
                    
                    reconnected = true;
                    OnConnected();
                    Console.WriteLine($"Reconnected to {_hostname}:{_port}");
                }
                catch (Exception ex)
                {
                    CleanupConnection();
                    Console.WriteLine($"Reconnection attempt {attempts} failed: {ex.Message}");
                }
            }
            
            return reconnected;
        }
        finally
        {
            _connectionLock.Release();
        }
    }:{port}");
            
            while (true)
            {
                Console.WriteLine("Waiting for a connection...");
                // Accept the incoming connection
                Socket handler = listener.Accept();
                
                // Process the connection in a separate task
                _ = Task.Run(() => ProcessClient(handler));
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error: {ex.Message}");
        }
        finally
        {
            // Ensure the socket is closed
            listener.Close();
        }
    }
    
    private async Task ProcessClient(Socket clientSocket)
    {
        try
        {
            string clientInfo = clientSocket.RemoteEndPoint.ToString();
            Console.WriteLine($"Client connected: {clientInfo}");
            
            // Buffer for incoming data
            byte[] buffer = new byte[1024];
            
            while (true)
            {
                // Receive data from the client
                int bytesReceived = await clientSocket.ReceiveAsync(buffer, SocketFlags.None);
                
                if (bytesReceived == 0)
                {
                    // Connection closed by the client
                    Console.WriteLine($"Client disconnected: {clientInfo}");
                    break;
                }
                
                // Convert the received bytes to a string
                string message = Encoding.ASCII.GetString(buffer, 0, bytesReceived);
                Console.WriteLine($"Received from {clientInfo}: {message}");
                
                // Echo the message back to the client
                byte[] responseBuffer = Encoding.ASCII.GetBytes($"Echo: {message}");
                await clientSocket.SendAsync(responseBuffer, SocketFlags.None);
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error handling client: {ex.Message}");
        }
        finally
        {
            // Close the client socket
            clientSocket.Close();
        }
    }
}
```

### Basic TCP Client with Socket API

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;

public class TcpSocketClient
{
    private Socket _clientSocket;
    
    public async Task ConnectAsync(string serverIp, int port)
    {
        try
        {
            // Create a TCP/IP socket
            _clientSocket = new Socket(AddressFamily.InterNetwork, 
                SocketType.Stream, ProtocolType.Tcp);
            
            // Connect to the server
            await _clientSocket.ConnectAsync(IPAddress.Parse(serverIp), port);
            Console.WriteLine($"Connected to server {serverIp}:{port}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Connection error: {ex.Message}");
            throw;
        }
    }
    
    public async Task<string> SendAndReceiveAsync(string message)
    {
        if (_clientSocket == null || !_clientSocket.Connected)
        {
            throw new InvalidOperationException("Client is not connected");
        }
        
        try
        {
            // Convert the string to a byte array
            byte[] messageBytes = Encoding.ASCII.GetBytes(message);
            
            // Send data to the server
            await _clientSocket.SendAsync(messageBytes, SocketFlags.None);
            
            // Receive response from the server
            byte[] buffer = new byte[1024];
            int bytesReceived = await _clientSocket.ReceiveAsync(buffer, SocketFlags.None);
            
            // Convert the response to a string
            return Encoding.ASCII.GetString(buffer, 0, bytesReceived);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Communication error: {ex.Message}");
            throw;
        }
    }
    
    public void Disconnect()
    {
        if (_clientSocket != null && _clientSocket.Connected)
        {
            // Close the socket
            _clientSocket.Shutdown(SocketShutdown.Both);
            _clientSocket.Close();
            Console.WriteLine("Disconnected from server");
        }
        
        _clientSocket = null;
    }
}
```

### Using Higher-Level TcpClient/TcpListener APIs

```csharp
using System;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;

public class TcpServerExample
{
    public async Task StartAsync(string ipAddress, int port)
    {
        // Create a TCP listener
        TcpListener listener = new TcpListener(IPAddress.Parse(ipAddress), port);
        
        try
        {
            // Start listening for client connections
            listener.Start();
            Console.WriteLine($"Server started on {ipAddress}:{port}");
            
            while (true)
            {
                Console.WriteLine("Waiting for a connection...");
                
                // Accept a client connection
                TcpClient client = await listener.AcceptTcpClientAsync();
                
                // Handle the client in a separate task
                _ = HandleClientAsync(client);
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Server error: {ex.Message}");
        }
        finally
        {
            // Stop the listener
            listener.Stop();
        }
    }
    
    private async Task HandleClientAsync(TcpClient client)
    {
        string clientInfo = client.Client.RemoteEndPoint.ToString();
        Console.WriteLine($"Client connected: {clientInfo}");
        
        try
        {
            // Get the network stream
            using NetworkStream stream = client.GetStream();
            using StreamReader reader = new StreamReader(stream);
            using StreamWriter writer = new StreamWriter(stream) { AutoFlush = true };
            
            while (client.Connected)
            {
                // Read data from the client
                string message = await reader.ReadLineAsync();
                
                if (message == null)
                {
                    // Client disconnected
                    break;
                }
                
                Console.WriteLine($"Received from {clientInfo}: {message}");
                
                // Send response back to the client
                await writer.WriteLineAsync($"Echo: {message}");
            }
        }
        catch (IOException)
        {
            // Client disconnected
            Console.WriteLine($"Client disconnected: {clientInfo}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error handling client {clientInfo}: {ex.Message}");
        }
        finally
        {
            // Close the client
            client.Close();
            Console.WriteLine($"Client connection closed: {clientInfo}");
        }
    }
}

public class TcpClientExample
{
    private TcpClient _client;
    private NetworkStream _stream;
    private StreamReader _reader;
    private StreamWriter _writer;
    
    public async Task ConnectAsync(string serverIp, int port)
    {
        try
        {
            // Create and connect the TCP client
            _client = new TcpClient();
            await _client.ConnectAsync(serverIp, port);
            
            // Get the network stream and create readers/writers
            _stream = _client.GetStream();
            _reader = new StreamReader(_stream);
            _writer = new StreamWriter(_stream) { AutoFlush = true };
            
            Console.WriteLine($"Connected to server {serverIp}:{port}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Connection error: {ex.Message}");
            Disconnect();
            throw;
        }
    }
    
    public async Task<string> SendMessageAsync(string message)
    {
        if (_client == null || !_client.Connected)
        {
            throw new InvalidOperationException("Client is not connected");
        }
        
        try
        {
            // Send the message
            await _writer.WriteLineAsync(message);
            
            // Read the response
            return await _reader.ReadLineAsync();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Communication error: {ex.Message}");
            Disconnect();
            throw;
        }
    }
    
    public void Disconnect()
    {
        _reader?.Dispose();
        _writer?.Dispose();
        _stream?.Dispose();
        _client?.Close();
        
        _reader = null;
        _writer = null;
        _stream = null;
        _client = null;
        
        Console.WriteLine("Disconnected from server");
    }
}
```

### Asynchronous Socket Programming

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Concurrent;

public class AsyncSocketServer
{
    private Socket _listener;
    private CancellationTokenSource _cts;
    private ConcurrentDictionary<string, Socket> _clients = new ConcurrentDictionary<string, Socket>();
    
    public async Task StartAsync(string ipAddress, int port, CancellationToken cancellationToken = default)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        
        // Create a TCP/IP socket
        _listener = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        
        try
        {
            // Bind and start listening
            _listener.Bind(new IPEndPoint(IPAddress.Parse(ipAddress), port));
            _listener.Listen(100);
            
            Console.WriteLine($"Server started on {ipAddress}:{port}");
            
            // Accept clients until cancellation is requested
            while (!_cts.Token.IsCancellationRequested)
            {
                try
                {
                    // Accept client asynchronously
                    Socket clientSocket = await Task.Run(() => _listener.Accept(), _cts.Token);
                    string clientId = Guid.NewGuid().ToString();
                    _clients.TryAdd(clientId, clientSocket);
                    
                    // Handle client in a separate task
                    _ = HandleClientAsync(clientId, clientSocket);
                }
                catch (OperationCanceledException)
                {
                    // Cancellation was requested
                    break;
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Error accepting client: {ex.Message}");
                }
            }
        }
        finally
        {
            await StopAsync();
        }
    }
    
    private async Task HandleClientAsync(string clientId, Socket clientSocket)
    {
        string clientInfo = clientSocket.RemoteEndPoint.ToString();
        Console.WriteLine($"Client connected: {clientInfo} (ID: {clientId})");
        
        try
        {
            // Create a buffer for receiving data
            byte[] buffer = new byte[8192];
            
            while (!_cts.Token.IsCancellationRequested && clientSocket.Connected)
            {
                // Create a receiving task with cancellation support
                var receiveTask = clientSocket.ReceiveAsync(buffer, SocketFlags.None);
                
                // Wait for data or cancellation
                int bytesRead;
                
                try
                {
                    bytesRead = await receiveTask.WaitAsync(_cts.Token);
                }
                catch (OperationCanceledException)
                {
                    break;
                }
                
                if (bytesRead == 0)
                {
                    // Client disconnected normally
                    break;
                }
                
                // Process received data
                string message = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                Console.WriteLine($"Received from {clientInfo}: {message}");
                
                // Echo back the message
                byte[] responseBuffer = Encoding.UTF8.GetBytes($"Echo: {message}");
                await clientSocket.SendAsync(responseBuffer, SocketFlags.None);
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error handling client {clientInfo}: {ex.Message}");
        }
        finally
        {
            // Clean up the client
            Console.WriteLine($"Client disconnected: {clientInfo} (ID: {clientId})");
            _clients.TryRemove(clientId, out _);
            
            try
            {
                clientSocket.Shutdown(SocketShutdown.Both);
            }
            catch
            {
                // Socket may already be closed
            }
            
            clientSocket.Close();
        }
    }
    
    public async Task StopAsync()
    {
        if (_cts != null && !_cts.IsCancellationRequested)
        {
            // Signal cancellation
            _cts.Cancel();
            
            // Close the listener
            if (_listener != null)
            {
                _listener.Close();
                _listener = null;
            }
            
            // Close all client connections
            foreach (var client in _clients)
            {
                try
                {
                    client.Value.Close();
                }
                catch
                {
                    // Ignore errors during cleanup
                }
            }
            
            _clients.Clear();
            Console.WriteLine("Server stopped");
            
            // Clean up the cancellation token source
            _cts.Dispose();
            _cts = null;
        }
        
        await Task.CompletedTask;
    }
}
```

## Working with Binary Data

```csharp
public static class BinaryProtocolExample
{
    public static async Task SendMessageAsync(Socket socket, Message message)
    {
        // Allocate a memory stream for the serialized message
        using var ms = new MemoryStream();
        using var writer = new BinaryWriter(ms);
        
        // Write message type (1 byte)
        writer.Write((byte)message.Type);
        
        // Write message ID (4 bytes)
        writer.Write(message.Id);
        
        // Write payload length (4 bytes)
        byte[] payloadBytes = Encoding.UTF8.GetBytes(message.Payload);
        writer.Write(payloadBytes.Length);
        
        // Write payload
        writer.Write(payloadBytes);
        
        // Get the buffer
        byte[] buffer = ms.ToArray();
        
        // Send the entire message
        await socket.SendAsync(buffer, SocketFlags.None);
    }
    
    public static async Task<Message> ReceiveMessageAsync(Socket socket)
    {
        // Header buffer - 9 bytes total (1 byte type + 4 bytes ID + 4 bytes length)
        byte[] headerBuffer = new byte[9];
        
        // Read the header
        int bytesRead = 0;
        while (bytesRead < headerBuffer.Length)
        {
            int received = await socket.ReceiveAsync(
                new ArraySegment<byte>(headerBuffer, bytesRead, headerBuffer.Length - bytesRead), 
                SocketFlags.None);
                
            if (received == 0)
                throw new EndOfStreamException("Connection closed while reading header");
                
            bytesRead += received;
        }
        
        // Parse the header
        byte messageType = headerBuffer[0];
        int messageId = BitConverter.ToInt32(headerBuffer, 1);
        int payloadLength = BitConverter.ToInt32(headerBuffer, 5);
        
        // Read the payload
        byte[] payloadBuffer = new byte[payloadLength];
        bytesRead = 0;
        
        while (bytesRead < payloadLength)
        {
            int received = await socket.ReceiveAsync(
                new ArraySegment<byte>(payloadBuffer, bytesRead, payloadLength - bytesRead), 
                SocketFlags.None);
                
            if (received == 0)
                throw new EndOfStreamException("Connection closed while reading payload");
                
            bytesRead += received;
        }
        
        // Convert payload to string
        string payload = Encoding.UTF8.GetString(payloadBuffer);
        
        // Create and return the message
        return new Message
        {
            Type = (MessageType)messageType,
            Id = messageId,
            Payload = payload
        };
    }
    
    public enum MessageType : byte
    {
        Request = 1,
        Response = 2,
        Notification = 3,
        Heartbeat = 4
    }
    
    public class Message
    {
        public MessageType Type { get; set; }
        public int Id { get; set; }
        public string Payload { get; set; }
    }
}
```

## Best Practices

### Socket Management

- **Resource Cleanup**: Always properly dispose of sockets, streams, and related resources
- **Graceful Disconnection**: Use `Socket.Shutdown()` before `Close()` for clean termination
- **Connection Pooling**: Reuse connections when possible instead of creating new ones
- **Backlog Configuration**: Set appropriate backlog size for `Listen()` based on expected load

```csharp
// Proper socket disposal
public void CloseConnection(Socket socket)
{
    if (socket != null && socket.Connected)
    {
        try
        {
            socket.Shutdown(SocketShutdown.Both);
            socket.Close();
        }
        catch (SocketException)
        {
            // Socket may already be closed
            socket.Close();
        }
    }
}
```

### Error Handling

- **Graceful Error Recovery**: Handle network exceptions without crashing the application
- **Timeouts**: Implement timeouts for all network operations
- **Connection Loss Detection**: Use keep-alive mechanisms or heartbeats
- **Retry Logic**: Implement exponential backoff for reconnection attempts

```csharp
public async Task<string> SendWithRetryAsync(string message, int maxRetries = 3)
{
    int retryCount = 0;
    TimeSpan delay = TimeSpan.FromSeconds(1);
    
    while (true)
    {
        try
        {
            return await _client.SendMessageAsync(message);
        }
        catch (Exception ex) when (ex is SocketException || ex is IOException)
        {
            if (++retryCount > maxRetries)
                throw;
                
            Console.WriteLine($"Communication failed, retrying in {delay.TotalSeconds}s: {ex.Message}");
            
            await Task.Delay(delay);
            delay = TimeSpan.FromMilliseconds(delay.TotalMilliseconds * 2); // Exponential backoff
            
            // Try to reconnect if needed
            if (!_client.Connected)
            {
                await _client.ConnectAsync(_serverIp, _serverPort);
            }
        }
    }
}
```

### Performance Optimization

- **Buffer Management**: Reuse buffers to reduce garbage collection
- **Buffer Sizing**: Use appropriately sized buffers (e.g., 4KB to 16KB)
- **Message Framing**: Implement proper message framing to handle incomplete reads
- **Asynchronous I/O**: Use async APIs for non-blocking operations
- **I/O Completion Ports**: Leverage the underlying OS's efficient I/O models

```csharp
// Reusable buffer pool example
public class BufferPool
{
    private readonly ConcurrentBag<byte[]> _buffers = new ConcurrentBag<byte[]>();
    private readonly int _bufferSize;
    private readonly int _maxBuffers;
    private int _count;
    
    public BufferPool(int bufferSize = 8192, int maxBuffers = 100)
    {
        _bufferSize = bufferSize;
        _maxBuffers = maxBuffers;
    }
    
    public byte[] Rent()
    {
        if (_buffers.TryTake(out byte[] buffer))
        {
            Interlocked.Decrement(ref _count);
            return buffer;
        }
        
        return new byte[_bufferSize];
    }
    
    public void Return(byte[] buffer)
    {
        if (buffer == null || buffer.Length != _bufferSize)
            return; // Don't store differently sized buffers
            
        if (Interlocked.Increment(ref _count) <= _maxBuffers)
        {
            _buffers.Add(buffer);
        }
        else
        {
            Interlocked.Decrement(ref _count);
            // Buffer is simply discarded and will be garbage collected
        }
    }
}
```

### Security

- **TLS/SSL**: Use secure connections for sensitive data
- **Connection Validation**: Implement authentication mechanisms
- **Data Validation**: Validate all incoming data before processing
- **Rate Limiting**: Implement throttling to prevent abuse

```csharp
// Example of setting up a secure SSL socket
public async Task<SslStream> CreateSecureClientConnectionAsync(
    string serverHost, 
    int port, 
    X509Certificate clientCertificate = null)
{
    TcpClient client = new TcpClient();
    await client.ConnectAsync(serverHost, port);
    
    SslStream sslStream = clientCertificate != null 
        ? new SslStream(
            client.GetStream(), 
            false, 
            ValidateServerCertificate, 
            null, 
            EncryptionPolicy.RequireEncryption)
        : new SslStream(
            client.GetStream(), 
            false, 
            ValidateServerCertificate, 
            null, 
            EncryptionPolicy.RequireEncryption);
    
    // The server name must match the name on the server certificate
    await sslStream.AuthenticateAsClientAsync(
        serverHost, 
        clientCertificate != null ? new X509CertificateCollection { clientCertificate } : null, 
        SslProtocols.Tls13, 
        true); // Check certificate revocation
    
    return sslStream;
}

private bool ValidateServerCertificate(
    object sender, 
    X509Certificate certificate, 
    X509Chain chain, 
    SslPolicyErrors sslPolicyErrors)
{
    // In production, implement proper certificate validation
    if (sslPolicyErrors == SslPolicyErrors.None)
        return true;
    
    Console.WriteLine($"Certificate error: {sslPolicyErrors}");
    
    // For development only - don't do this in production
    // return true;
    
    // For production
    return false;
}
```

## Common Pitfalls

### Blocking I/O Operations

**Problem**: Using synchronous I/O in a UI or server application can freeze the UI or limit scalability.

**Solution**: Use asynchronous methods (Task-based) for all I/O operations.

```csharp
// AVOID this blocking approach
public string GetResponse(string message)
{
    byte[] buffer = new byte[1024];
    _socket.Send(Encoding.ASCII.GetBytes(message));  // Blocks
    int received = _socket.Receive(buffer);  // Blocks
    return Encoding.ASCII.GetString(buffer, 0, received);
}

// PREFER this non-blocking approach
public async Task<string> GetResponseAsync(string message)
{
    byte[] buffer = new byte[1024];
    await _socket.SendAsync(Encoding.ASCII.GetBytes(message), SocketFlags.None);
    int received = await _socket.ReceiveAsync(buffer, SocketFlags.None);
    return Encoding.ASCII.GetString(buffer, 0, received);
}
```

### Socket Resource Leaks

**Problem**: Not closing sockets properly leads to resource exhaustion.

**Solution**: Use `using` statements or try/finally blocks to ensure cleanup.

```csharp
// AVOID resources not being cleaned up
public void ProcessClient(TcpClient client)
{
    NetworkStream stream = client.GetStream();
    // Process the client...
    // If an exception occurs here, resources might not be released
}

// PREFER proper resource disposal
public void ProcessClient(TcpClient client)
{
    using (client)
    using (NetworkStream stream = client.GetStream())
    {
        // Process the client...
    }
    // Resources are always released, even if exceptions occur
}
```

### Incomplete Message Reads

**Problem**: Assuming a single `Receive` call will read an entire message.

**Solution**: Implement proper message framing or buffering.

```csharp
// AVOID assuming a single read gets the full message
public string ReceiveMessage(Socket socket)
{
    byte[] buffer = new byte[1024];
    int received = socket.Receive(buffer);
    return Encoding.ASCII.GetString(buffer, 0, received);
    // This might return partial messages if they're larger than the buffer
}

// PREFER handling message boundaries properly
public async Task<string> ReceiveFullMessageAsync(Socket socket)
{
    // First read the message length (4 bytes)
    byte[] headerBuffer = new byte[4];
    int bytesRead = 0;
    
    while (bytesRead < 4)
    {
        int received = await socket.ReceiveAsync(
            new ArraySegment<byte>(headerBuffer, bytesRead, 4 - bytesRead), 
            SocketFlags.None);
            
        if (received == 0)
            throw new EndOfStreamException("Connection closed while reading header");
            
        bytesRead += received;
    }
    
    // Get the message length
    int messageLength = BitConverter.ToInt32(headerBuffer, 0);
    
    // Now read the actual message data
    byte[] messageBuffer = new byte[messageLength];
    bytesRead = 0;
    
    while (bytesRead < messageLength)
    {
        int received = await socket.ReceiveAsync(
            new ArraySegment<byte>(messageBuffer, bytesRead, messageLength - bytesRead), 
            SocketFlags.None);
            
        if (received == 0)
            throw new EndOfStreamException("Connection closed while reading message");
            
        bytesRead += received;
    }
    
    return Encoding.ASCII.GetString(messageBuffer);
}
```

### Not Handling Connection Drops

**Problem**: Failing to detect or handle network disconnections gracefully.

**Solution**: Implement heartbeats and proper error detection.

```csharp
public class HeartbeatManager
{
    private readonly Socket _socket;
    private readonly Timer _timer;
    private readonly TimeSpan _interval;
    private readonly TimeSpan _timeout;
    private DateTime _lastHeartbeatReceived;
    
    public event EventHandler ConnectionLost;
    
    public HeartbeatManager(Socket socket, TimeSpan interval, TimeSpan timeout)
    {
        _socket = socket;
        _interval = interval;
        _timeout = timeout;
        _lastHeartbeatReceived = DateTime.UtcNow;
        
        // Create a timer to send heartbeats and check connection
        _timer = new Timer(HeartbeatCallback, null, TimeSpan.Zero, interval);
    }
    
    private void HeartbeatCallback(object state)
    {
        try
        {
            // Check if we've received a heartbeat within the timeout period
            if (DateTime.UtcNow - _lastHeartbeatReceived > _timeout)
            {
                // Connection considered lost
                _timer.Change(Timeout.Infinite, Timeout.Infinite);
                ConnectionLost?.Invoke(this, EventArgs.Empty);
                return;
            }
            
            // Send a heartbeat
            byte[] heartbeat = Encoding.ASCII.GetBytes("PING");
            _socket.Send(heartbeat);
        }
        catch
        {
            // Error sending heartbeat, connection is probably lost
            _timer.Change(Timeout.Infinite, Timeout.Infinite);
            ConnectionLost?.Invoke(this, EventArgs.Empty);
        }
    }
    
    public void HeartbeatReceived()
    {
        _lastHeartbeatReceived = DateTime.UtcNow;
    }
    
    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

### Non-thread-safe Socket Access

**Problem**: Accessing sockets from multiple threads without synchronization.

**Solution**: Use synchronization mechanisms or ensure single-threaded access.

```csharp
public class ThreadSafeSocket
{
    private readonly Socket _socket;
    private readonly SemaphoreSlim _sendLock = new SemaphoreSlim(1, 1);
    private readonly SemaphoreSlim _receiveLock = new SemaphoreSlim(1, 1);
    
    public ThreadSafeSocket(Socket socket)
    {
        _socket = socket;
    }
    
    public async Task SendAsync(byte[] buffer)
    {
        await _sendLock.WaitAsync();
        try
        {
            await _socket.SendAsync(buffer, SocketFlags.None);
        }
        finally
        {
            _sendLock.Release();
        }
    }
    
    public async Task<int> ReceiveAsync(byte[] buffer)
    {
        await _receiveLock.WaitAsync();
        try
        {
            return await _socket.ReceiveAsync(buffer, SocketFlags.None);
        }
        finally
        {
            _receiveLock.Release();
        }
    }
    
    public void Dispose()
    {
        _sendLock.Dispose();
        _receiveLock.Dispose();
    }