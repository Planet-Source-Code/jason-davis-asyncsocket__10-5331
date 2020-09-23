<div align="center">

## AsyncSocket


</div>

### Description

Simplifies using asynchronous sockets in .Net - kind of resemble the old WinSock control for VB6. Using this class as a listener for multithreaded applications works GREAT! I use it in my email server (http://sourceforge.net/projects/ilkmail/). I spent a lot of time on this very small class to make it work the best it can, so if you like it, please vote for me! Also, any comments, suggestions, code fixes, etc would be VERY welcome!
 
### More Info
 


<span>             |<span>
---                |---
**Submitted On**   |
**By**             |[Jason Davis](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByAuthor/jason-davis.md)
**Level**          |Intermediate
**User Rating**    |4.3 (17 globes from 4 users)
**Compatibility**  |C\#
**Category**       |[Miscellaneous](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByCategory/miscellaneous__10-1.md)
**World**          |[\.Net \(C\#, VB\.net\)](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByWorld/net-c-vb-net.md)
**Archive File**   |[](https://github.com/Planet-Source-Code/jason-davis-asyncsocket__10-5331/archive/master.zip)





### Source Code

/* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *
* AsyncSocket                       *
* Copyright (C) 2006 ILK Technologies                      *
*                                        *
* This program is free software; you can redistribute it and/or modify      *
* it under the terms of the GNU General Public License as published by      *
* the Free Software Foundation; either version 2 of the License, or       *
* (at your option) any later version.                      *
*                                        *
* This program is distributed in the hope that it will be useful,        *
* but WITHOUT ANY WARRANTY; without even the implied warranty of         *
* MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the         *
* GNU General Public License for more details.                  *
*                                        *
* You should have received a copy of the GNU General Public License       *
* along with this program; if not, write to the Free Software          *
* Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA *
* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * */
/* Remember the good old days of the VB Winsock control?
 * Very limited, but very simple. I made great use of it
 * for standard TCP/IP connections. This is my attempt to
 * recreate it in .Net, with some minor changes. I use
 * IO completion ports for asynchronous communication so
 * applications don't block while waiting for events.
 * If an app wants blocking mode, use the TCPClient class
 * in the System.Net.Sockets namespace.
 */
using System;
using System.IO;
using System.Net;
using System.Net.Security;
using System.Security.Authentication;
using System.Security.Cryptography.X509Certificates;
using System.Net.Sockets;
using System.Text;
using System.Threading;
namespace ILKTechnologies
{
  /// <summary>
  /// Handles underlying code for asynchrounous socket communications.
  /// </summary>
  public class AsyncSocket : IDisposable
  {
    Socket _socket;           //Socket object
    Stream _stream;           //By using a base stream, we can call the same code for NetworkStream AND SslStream
    byte[] _buffer = new byte[1024];  //Receive Buffer
    DateTime _lastActivity;       //Holds a marker for the timeout timer
    Timer _timer;            //Timeout timer
    double _timeout = 60;        //Number of seconds before timeout occures
    bool _raiseTimeout = false;     //Weather timeout events are triggered
    //Event delegates
    public delegate void OnConnectEvent();
    public delegate void OnExceptionEvent(Exception Ex);
    public delegate void OnAcceptEvent(Socket Client);
    public delegate void OnReceiveEvent(byte[] Data);
    public delegate void OnSendCompleteEvent();
    public delegate void OnDisconnectEvent();
    public delegate void OnSecureEvent();
    public delegate void OnTimeoutEvent();
    //Events
    public event OnConnectEvent OnConnect;
    public event OnExceptionEvent OnException;
    public event OnAcceptEvent OnAccept;
    public event OnReceiveEvent OnReceive;
    public event OnSendCompleteEvent OnSendComplete;
    public event OnDisconnectEvent OnDisconnect;
    public event OnSecureEvent OnSecure;
    public event OnTimeoutEvent OnTimeout;
    public AsyncSocket()
    {
      Reset();
    }
    public AsyncSocket(Socket Connection)
    {
      //Setup the vars
      Reset();
      _lastActivity = DateTime.Now;
      _timer = new Timer(new TimerCallback(TimerCallback), null, 1000, 1000);
      _socket = Connection;
      //Get the stream and wait for data
      NetworkStream ns = new NetworkStream(_socket);
      _stream = (Stream)ns;
      _stream.BeginRead(_buffer, 0, _buffer.Length, new AsyncCallback(ReadCallback), null);
    }
    public IPEndPoint LocalEndPoint
    {
      get { return (IPEndPoint)_socket.LocalEndPoint; }
    }
    public IPEndPoint RemoteEndPoint
    {
      get { return (IPEndPoint)_socket.RemoteEndPoint; }
    }
    /// <summary>
    /// Gets or sets the number of seconds to wait before triggering the timeout event.
    /// </summary>
    public double Timeout
    {
      get { return _timeout; }
      set { _timeout = value; }
    }
    /// <summary>
    /// Gets or sets weather the timeout event is triggered.
    /// </summary>
    public bool RaiseTimeout
    {
      get { return _raiseTimeout; }
      set { _raiseTimeout = value; }
    }
    /// <summary>
    /// Closes the socket and releases all resources
    /// </summary>
    public void Dispose()
    {
      try { _stream.Dispose(); }
      catch { }
      finally { _stream = null; }
      try { _socket.Close(); }
      catch { }
      finally { _socket = null; }
      try { _timer.Dispose(); }
      catch { }
      finally { _timer = null; }
    }
    /// <summary>
    /// Resets the socket and timeout timer
    /// </summary>
    public void Reset()
    {
      try { _stream.Close(); }
      catch { }
      try { _stream.Dispose(); }
      catch { }
      try { _socket.Close(); }
      catch { }
      try { _timer.Dispose(); }
      catch { }
      try { _socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp); }
      catch { }
    }
    /// <summary>
    /// Begins listening for incoming connections
    /// </summary>
    /// <param name="LocalAddress"></param>
    /// <param name="Port"></param>
    public void Listen(IPAddress LocalAddress, int Port)
    {
      IPEndPoint ipep = new IPEndPoint(LocalAddress, Port);
      Listen(ipep);
    }
    /// <summary>
    /// Begins listening for incoming connections
    /// </summary>
    /// <param name="Port">The port to listen to</param>
    public void Listen(int Port)
    {
      IPEndPoint ipep = new IPEndPoint(IPAddress.Any, Port);
      Listen(ipep);
    }
    /// <summary>
    /// Begins listening for incoming connections
    /// </summary>
    /// <param name="LocalEndpoint"></param>
    public void Listen(IPEndPoint LocalEndpoint)
    {
      //Set the socket to listenn and begin accepting connections
      _socket.Bind(LocalEndpoint);
      _socket.Listen((int)SocketOptionName.MaxConnections);
      _socket.BeginAccept(new AsyncCallback(AcceptCallback), null);
    }
    /// <summary>
    /// Begins connecting to a remote computer
    /// </summary>
    /// <param name="RemoteHost"></param>
    /// <param name="Port"></param>
    public void Connect(string RemoteHost, int Port)
    {
      _socket.BeginConnect(RemoteHost, Port, new AsyncCallback(ConnectCallback), null);
    }
    /// <summary>
    /// Begins connecting to a remote computer
    /// </summary>
    /// <param name="RemoteEndpoint"></param>
    public void Connect(IPEndPoint RemoteEndpoint)
    {
      _socket.BeginConnect(RemoteEndpoint, new AsyncCallback(ConnectCallback), null);
    }
    /// <summary>
    /// Begins connecting to a remote computer
    /// </summary>
    /// <param name="RemoteAddress"></param>
    /// <param name="Port"></param>
    public void Connect(IPAddress RemoteAddress, int Port)
    {
      _socket.BeginConnect(RemoteAddress, Port, new AsyncCallback(ConnectCallback), null);
    }
    /// <summary>
    /// Begins connecting to a remote computer
    /// </summary>
    /// <param name="RemoteAddresses"></param>
    /// <param name="Port"></param>
    public void Connect(IPAddress[] RemoteAddresses, int Port)
    {
      _socket.BeginConnect(RemoteAddresses, Port, new AsyncCallback(ConnectCallback), null);
    }
    /// <summary>
    /// Begins securing the socket using Ssl
    /// </summary>
    /// <param name="Certificate"></param>
    /// <param name="ClientCertificateRequired"></param>
    /// <param name="Protocol"></param>
    /// <param name="CheckRevocation"></param>
    public void SecureSocket(X509Certificate Certificate, bool ClientCertificateRequired, SslProtocols Protocol, bool CheckRevocation)
    {
      _lastActivity = DateTime.Now;
      SslStream ss = new SslStream(_stream);
      ss.BeginAuthenticateAsServer(Certificate, ClientCertificateRequired, Protocol, CheckRevocation, new AsyncCallback(AuthenticateCallback), null);
    }
    /// <summary>
    /// Sends data to the remote host
    /// </summary>
    /// <param name="Data"></param>
    public void Send(byte[] Data)
    {
      _lastActivity = DateTime.Now;
      _stream.BeginWrite(Data, 0, Data.Length, new AsyncCallback(WriteCallback), null);
    }
    /// <summary>
    /// Sends data to the remote host
    /// </summary>
    /// <param name="Data"></param>
    public void Send(string Data)
    {
      Send(Encoding.ASCII.GetBytes(Data));
    }
    /// <summary>
    /// Begins closing the connection
    /// </summary>
    public void Disconnect()
    {
      _lastActivity = DateTime.Now;
      _socket.Shutdown(SocketShutdown.Both);
      _socket.BeginDisconnect(true, new AsyncCallback(DisconnectCallback), null);
    }
    private void DisconnectCallback(IAsyncResult ar)
    {
      try
      {
        _socket.EndDisconnect(ar);
        if (OnDisconnect != null && OnDisconnect.GetInvocationList().Length > 0)
          OnDisconnect.Invoke();
      }
      catch (Exception Ex)
      {
        if (OnException != null && OnException.GetInvocationList().Length > 0)
          OnException.Invoke(Ex);
      }
    }
    private void ConnectCallback(IAsyncResult ar)
    {
      _lastActivity = DateTime.Now;
      _timer = new Timer(new TimerCallback(TimerCallback), null, 1000, 1000);
      try
      {
        _socket.EndConnect(ar);
        if (OnConnect != null && OnConnect.GetInvocationList().Length > 0)
          OnConnect.Invoke();
        NetworkStream ns = new NetworkStream(_socket, true);
        _stream = (Stream)ns;
        _stream.BeginRead(_buffer, 0, _buffer.Length, new AsyncCallback(ReadCallback), null);
      }
      catch (Exception Ex)
      {
        if (OnException != null && OnException.GetInvocationList().Length > 0)
          OnException.Invoke(Ex);
      }
    }
    private void AcceptCallback(IAsyncResult ar)
    {
      try
      {
        if (OnAccept != null && OnAccept.GetInvocationList().Length > 0)
          OnAccept.Invoke(_socket.EndAccept(ar));
        _socket.BeginAccept(new AsyncCallback(AcceptCallback), null);
      }
      catch (Exception Ex)
      {
        if (OnException != null && OnException.GetInvocationList().Length > 0)
          OnException.Invoke(Ex);
      }
    }
    private void ReadCallback(IAsyncResult ar)
    {
      _lastActivity = DateTime.Now;
      try
      {
        int size = _stream.EndRead(ar);
        byte[] data = new byte[size];
        Array.Copy(_buffer, data, size);
        if (OnReceive != null && OnReceive.GetInvocationList().Length > 0)
          OnReceive.Invoke(data);
        _stream.BeginRead(_buffer, 0, _buffer.Length, new AsyncCallback(ReadCallback), null);
      }
      catch (Exception Ex)
      {
        if (OnException != null & OnException.GetInvocationList().Length > 0)
          OnException.Invoke(Ex);
      }
    }
    private void WriteCallback(IAsyncResult ar)
    {
      _lastActivity = DateTime.Now;
      try
      {
        _stream.EndWrite(ar);
        if (OnSendComplete != null && OnSendComplete.GetInvocationList().Length > 0)
          OnSendComplete.Invoke();
      }
      catch (Exception Ex)
      {
        if (OnException != null & OnException.GetInvocationList().Length > 0)
          OnException.Invoke(Ex);
      }
    }
    private void AuthenticateCallback(IAsyncResult ar)
    {
      _lastActivity = DateTime.Now;
      try
      {
        SslStream ss = (SslStream)ar.AsyncState;
        ss.EndAuthenticateAsServer(ar);
        if (OnSecure != null && OnSecure.GetInvocationList().Length > 0)
          OnSecure.Invoke();
        _stream = (Stream)ss;
        _stream.BeginRead(_buffer, 0, _buffer.Length, new AsyncCallback(ReadCallback), null);
      }
      catch (Exception Ex)
      {
        if (OnException != null && OnException.GetInvocationList().Length > 0)
          OnException.Invoke(Ex);
      }
    }
    private void TimerCallback(object State)
    {
      if (_lastActivity.AddSeconds(_timeout) <= DateTime.Now)
        if (_raiseTimeout)
        {
          if (OnTimeout != null && OnTimeout.GetInvocationList().Length > 0)
            OnTimeout.Invoke();
          Reset();
        }
    }
  }
}

