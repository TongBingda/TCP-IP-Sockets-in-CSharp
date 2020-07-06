# 二、基本套接字

现在您准备好学习用C＃编写自己的套接字应用程序。C＃编程语言的优点之一是使用Microsoft的.NET框架，该框架提供了强大的API编程库。在提供的类库中，有System.Net和System.Net.Sockets命名空间，并且本书的大部分内容专门致力于如何使用那里提供的套接字API。在本章中，我们首先说明C＃应用程序如何识别网络主机。然后，我们描述TCP和UDP客户端和服务器的创建。.NET框架提供了在使用TCP和UDP之间的明显区别，为这两种协议定义了一组单独的类，因此我们将它们分开对待。最后，我们讨论Socket类，它是所有更高级别的.NET套接字类的基础实现。

## 2.1 套接字地址

IPv4使用32位二进制地址来识别通信主机。客户端在启动通信时必须指定运行服务器程序的主机的IP地址。网络基础结构使用32位目标地址将客户的信息路由到适当的计算机。地址可以用C＃中的32位长整数值指定，也可以使用包含数字地址的点分四进制表示形式的字符串（例如169.1.1.1）来指定。.NET将IP地址抽象封装在IPAddress类中，该类可以在其构造函数中使用长整数IP参数，或者使用其Parse()方法以IP地址的点分四进制表示形式处理字符串。Dns类还提供了一种机制来查找或解析IP地址的名称（例如server.example.com）。由于在现代Internet中，单个服务器解析为多个IP地址或名称别名并不罕见，因此将结果返回到容器类IPHostEntry中，该容器类包含一个或多个字符串主机名和IPAddress类实例的数组。

Dns类具有几种解析IP地址的方法。GetHostName()方法不带任何参数，并返回包含本地主机名的字符串。GetHostByName()和Resolve()方法基本相同。它们采用包含要查找的主机名的字符串参数，并以IPHostEntry类实例的形式返回所提供输入的IP地址和主机名信息。GetHostByAddress()方法采用一个字符串参数，其中包含IP地址的点分四进制字符串表示形式，并且还返回IPHostEntry实例中的主机信息。

我们的第一个程序示例IPAddressExample.cs演示了Dns，IPAddress和IPHostEntry类的用法。该程序将名称或IP地址的列表作为命令行参数，并打印本地主机的名称和IP地址，然后打印在命令行上指定的主机的名称和IP地址。

```csharp
using System; // For String and Console
using System.Net; // For Dns, IPHostEntry, IPAddress
using System.Net.Sockets; // For SocketException

namespace ConsoleApplication1
{
    class Program
    {
        static void PrintHostInfo(String host)
        {
            try
            {
                IPHostEntry hostInfo;
                // Attempt to resolve DNS for given host or address
                hostInfo = Dns.Resolve(host);
                // Display list of IP addresses for this host
                Console.Write("\tIP Addresses: ");
                foreach (IPAddress ipaddr in hostInfo.AddressList)
                {
                    Console.Write(ipaddr.ToString() + " ");
                }
                Console.WriteLine();
                // Display list of alias names for this host
                Console.Write("\tAliases: ");
                foreach (String alias in hostInfo.Aliases)
                {
                    Console.Write(alias + " ");
                }
                Console.WriteLine("\n");
            }
            catch (Exception)
            {
                Console.WriteLine("\tUnable to resolve host:" + host + "\n");
            }
        }
        static void Main(string[] args)
        {
            // Get and print local host info
            try
            {
                Console.WriteLine("Local Host:");
                String localHostName = Dns.GetHostName();
                Console.WriteLine("\tHost Name: " + localHostName);
                PrintHostInfo(localHostName);
            }
            catch (Exception)
            {
                Console.WriteLine("Unable to resolve local host\n");
            }
            // Get and print info for hosts given on command line
            foreach (String arg in args)
            {
                Console.WriteLine(arg + ":");
                PrintHostInfo(arg);
            }
        }
    }
}
```

## 2.2 .NET中的套接字实现

在我们开始描述.NET套接字类的详细信息之前，对Microsoft Windows上的套接字进行简要概述和历史记录很有用。套接字最初是为UNIX的Berkeley软件发行版（BSD）创建的。用于Microsoft Windows的套接字版本WinSock 1.1最初于1992年发布，当前版本为2.0。除了一些细微的差异外，WinSock提供了Berkeley套接字C接口中可用的标准套接字功能（本书的C版本详细描述了该接口）。

2002年，Microsoft发布了称为.NET的标准化API框架，该框架提供了Microsoft提供的所有编程语言的统一类库。该库的功能包括较高级别的类，这些类隐藏了许多实现细节并简化了许多编程任务。但是，抽象有时会隐藏较低级别接口的某些灵活性和功能。为了允许访问底层的套接字接口，Microsoft实现了.NET Socket类，该类是WinSock套接字函数的包装，并且公开了套接字接口的大多数通用性（和复杂性）。然后，使用.NET套接字包装器类实现了三个更高级别的套接字类TcpClient，TcpListener和UdpClient。实际上，这些类具有受保护的属性，该属性是它们正在使用的Socket类的实例。如图2.1所示。



图2.1 套接字类的关系

为什么要知道这一点很重要？ 首先，要澄清我们指的是“socket”的含义。socket一词在网络编程中已经具有许多不同的含义，从API到类名或实例。通常，当我们引用大写的“Socket”时，是指.NET类，而小写的“socket”是指使用任何.NET套接字类的套接字实例。

其次，底层实现有时对.NET程序员来说很明显。有时需要利用Socket类来利用高级功能。基础WinSock实现的某些组件仍然可见，例如WinSock错误代码的使用，可通过SocketException的ErrorCode属性获得这些错误代码，这些代码可用于确切确定发生了什么类型的错误。WinSock错误代码在附录中有更详细的讨论。

## 2.3 TCP套接字

.NET框架专门为TCP提供了两个类：TcpClient和TcpListener。这些类提供了Socket类的更高级别的抽象，但是正如我们将看到的，只有通过直接使用Socket类才能使用高级功能。

### 2.3.1 TCP客户端
TCP客户端启动与服务器的通信，该服务器被动等待被联系。典型的TCP客户端经历三个步骤：

1.构造TcpClient的实例：可以通过指定远程主机和端口，或使用Connect()方法显式地在构造函数中隐式创建TCP连接。

2.使用套接字的流进行通信：TcpClient的连接实例包含一个NetworkStream，可以像使用任何其他.NET I/O流一样使用。

3.关闭连接：调用TcpClient的Close()方法。

我们的第一个TCP应用程序称为TcpEchoClient.cs，它是使用TCP与回显服务器通信的客户端。回显服务器只是简单地将其收到的任何内容重复发送回客户端。要回显的字符串作为命令行参数提供给我们的客户端。许多系统都包含用于调试和测试目的的回显服务器。要测试标准回显服务器是否正在运行，请尝试远程登录到服务器上的端口7（默认回显端口）（例如，在命令行“telnet server.example.com 7”或使用您选择的telnet应用程序）。如果不是，则可以从下一部分针对TcpEchoServer.cs服务器运行此客户端。

```csharp
using System; // For String, Int32, Console, ArgumentException
using System.Text; // For Encoding
using System.IO; // For IOException
using System.Net.Sockets; // For SocketException

namespace ConsoleApplication1
{
    class Program
    {
        static void Main(string[] args)
        {
            if ((args.Length < 2) || (args.Length > 3)) // Test for correct # of args
            {
                throw new ArgumentException("Parameters: <Server> <Word> [<Port>]");
            }
            String server = args[0]; // Server name or IP address
            // Convert input String to bytes
            byte[] byteBuffer = Encoding.ASCII.GetBytes(args[1]);
            // Use port argument if supplied, otherwise default to 7
            int servPort = (args.Length == 3) ? Int32.Parse(args[2]) : 7;
            TcpClient client = null;
            NetworkStream netStream = null;
            try 
            {
                // Create socket that is connected to server on specified port
                client = new TcpClient(server, servPort);
                Console.WriteLine("Connected to server... sending echo string");
                netStream = client.GetStream();
                // Send the encoded string to the server
                netStream.Write(byteBuffer, 0, byteBuffer.Length);
                Console.WriteLine("Sent {0} bytes to server...", byteBuffer.Length);
                int totalBytesRcvd = 0; // Total bytes received so far
                int bytesRcvd = 0; // Bytes received in last read
                // Receive the same string back from the server
                while (totalBytesRcvd < byteBuffer.Length)
                {
                    if ((bytesRcvd = netStream.Read(byteBuffer, totalBytesRcvd, byteBuffer.Length - totalBytesRcvd)) == 0)
                    {
                        Console.WriteLine("Connection closed prematurely.");
                        break;
                    }
                    totalBytesRcvd += bytesRcvd;
                }
                Console.WriteLine("Received {0} bytes from server: {1}", totalBytesRcvd, Encoding.ASCII.GetString(byteBuffer, 0, totalBytesRcvd));
            }
            catch (Exception e)
            {
                Console.WriteLine(e.Message);
            }
            finally
            {
                netStream.Close();
                client.Close();
            }
        }
    }
}
```

### 2.3.2 TCP服务器

现在，我们将注意力转移到构建服务器上。服务器的工作是为客户端设置一个端点，以使其连接并被动等待连接。典型的TCP服务器经历两个步骤：

1.构造一个TcpListener实例，指定本地地址和端口，然后调用Start()方法。该套接字侦听指定端口上的传入连接。

2.反复：

■调用TcpListener的AcceptTcpClient()方法来获取下一个传入的客户端连接。建立新的客户端连接后，将为AcceptTcpClient()调用创建并返回用于新连接的TcpClient实例。

■使用TcpClient的NetworkStream的Read()和Write()方法与客户端进行通信。

■使用NetworkStream和TcpClient的Close()方法关闭新的客户端套接字连接和流。

请注意，在C＃中，无论在客户端还是服务器中，TcpClient类都用于访问TCP连接。 可以使用同一类，因为TCP协议实际上不会区分客户端和服务器，尤其是在建立连接之后。

作为AcceptTcpClient()的替代方法，TcpListener类还具有一个AcceptSocket()方法，该方法返回传入客户端连接的Socket实例。套接字类将在2.5节中详细介绍。

我们的下一个示例TcpEchoServer.cs实现了客户端程序使用的回显服务。服务器非常简单。它永远运行，反复接受连接，接收并回显字节，直到客户端关闭连接，然后关闭客户端套接字。

```csharp
using System; // For Console, Int32, ArgumentException, Environment
using System.Net; // For IPAddress
using System.Net.Sockets; // For TcpListener, TcpClient

namespace ConsoleApplication1
{
    class Program
    {
        private const int BUFSIZE = 32; // Size of receive buffer
        static void Main(string[] args)
        {
            if (args.Length > 1) // Test for correct # of args
                throw new ArgumentException("Parameters: [<Port>]");
            int servPort = (args.Length == 1) ? Int32.Parse(args[0]) : 7;
            TcpListener listener = null;
            try
            {
                // Create a TCPListener to accept client connections
                listener = new TcpListener(IPAddress.Any, servPort);
                listener.Start();
            }
            catch (SocketException se)
            {
                Console.WriteLine(se.ErrorCode + ": " + se.Message);
                Environment.Exit(se.ErrorCode);
            }
            byte[] rcvBuffer = new byte[BUFSIZE]; // Receive buffer
            int bytesRcvd; // Received byte count
            for (; ; ) // Run forever, accepting and servicing connections
            {
                TcpClient client = null;
                NetworkStream netStream = null;
                try
                {
                    client = listener.AcceptTcpClient(); // Get client connection
                    netStream = client.GetStream();
                    Console.Write("Handling client - ");
                    // Receive until client closes connection, indicated by 0 return value
                    int totalBytesEchoed = 0;
                    while ((bytesRcvd = netStream.Read(rcvBuffer, 0, rcvBuffer.Length)) > 0)
                    {
                        netStream.Write(rcvBuffer, 0, bytesRcvd);
                        totalBytesEchoed += bytesRcvd;
                    }
                    Console.WriteLine("echoed {0} bytes.", totalBytesEchoed);
                    // Close the stream and socket. We are done with this client!
                    netStream.Close();
                    client.Close();
                }
                catch (Exception e)
                {
                    Console.WriteLine(e.Message); 
                    netStream.Close();
                }
            }
        }
    }
}
```

### 2.3.3 流

如前面的示例所示，.NET框架中I/O的主要范例是流抽象。流只是字节的有序序列。.NET流支持将字节读取和写入流。在我们的TCP客户端和服务器中，每个TcpClient或TcpListener实例都包含一个NetworkStream实例。当我们写入TcpClient的流时，字节（最终）可以从连接另一端的TcpListener的流中读取。Socket和UdpClient类使用字节数组而不是流来发送和接收数据。如果读取或写入错误，则NetworkStream将引发IOException。有关流的更多详细信息，请参见第3.2节。
