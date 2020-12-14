
---
title: Winsock创建socket server和client
date: 2020-12-14 20:37:14
author: 马捷径
img: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=105906878,390165426&fm=26&gp=0.jpg
top: true
cover: true
coverImg: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=105906878,390165426&fm=26&gp=0.jpg
toc: true
mathjax: true
summary: 深度理解在Win10上利用Winsock创建socket的服务端和客户端。
keywords: 
  - 通信 
  - socket
  - Winsock
  - C++
categories: Communication
tags:
  - socket
---



本文详细介绍在Windows 10下利用Winsock创建socket server和client应用步骤和源码。项目源码可以到我的[github](https://github.com/MolianWH/CommutionTools/tree/main/Socket/SocketWin)上下载。在该仓库中，我准备将所有有关通信方式的源码做成工具包，便于以后开发直接使用。本文还可以在我的[CSND](https://blog.csdn.net/weixin_38369492/article/details/111032276)上查看。

---


## 1. 基本流程

创建TCP/IP流的server/client通用步骤如下：

### 1.1 Server和Client创建流程
**Server**
1. 初始化Winsock
2. 创建socket
3. 绑定socket
4. 监听客户端socket
5. 接受客户端连接请求
6. 接收和发送数据
7.  断开连接

**Client**
1. 初始化Winsock
2. 创建socket
3. 连接服务端
4. 发送和接收数据
5. 断开连接

### 1.2 创建Winsock应用步骤
创建一个最基础的Winsock应用需要以下几步

1. 创建一个空项目
2. 添加一个空的C++ source文件到项目中
3. 引用Microsoft Windows SDK 的Include、Lib和Src目录
4. 确保项目连接Winsock库文件：`#pragma comment(lib, Ws2_32.lib)`
5. 编写Winsock应用程序。使用Winsock API需要包含两个头文件：**Winsock2.h**和**Ws2tcpip.h**。前者包含Winsock的大多数函数、结构体、定义；后者包含在WinSock 2中关于TCP/IP协议的用于检索IP地址的新函数和结构。

通常一个Winsock应用的头部应该这样写：
```cpp
#include <WinSock2.h>
#include <WS2tcpip.h>
#include <stdio.h>

#pragma comment(lib, "Ws2_32.lib")

int main() {
  return 0;
}
```

> **Note**
> - 如果使用ip helper APIs，需要包含Iphlpapi.h。且WinSock2.h需要在其前面。
> - Winsock2.h包含了Windows.h一些核心内容，所以通常不需要再包含Windows.h了
> - 如果要包含Windows.h，必须放在Winsock2.h前，并且要使用`#define WIN32_LEAN_AND_MEAN`。这是因为Windows.h中包含了Winsock.h（第一个版本），会和Winsock2.h冲突，使用该预定义会避免用Winsock.h

所以一个升级版的头部应该这样写：

```cpp
#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif // !WIN32_LEAN_AND_MEAN

#include <Windows.h>
#include <WinSock2.h>  // socket
#include <WS2tcpip.h>  // TCP/IP
#include <iphlpapi.h>  // ip helper APIs
#include <stdio.h>

// Link to Ws2_32.lib
#pragma comment(lib,"Ws2_32.lib")

int main() {
  return 0;
}
```

### 1.3 初始化Winsock
所有调用Winsock函数的进程(应用程序或DLL)必须在调用其他Winsock函数之前初始化Windows socket DLL再使用。这也确保了系统上支持Winsock。

1. 创建WSADATA对象
```cpp
   WSADATA wsaData;
```
2. 调用WSAStartup，返回整数值，并通过该值检查错误。 
 ```cpp
    // Initialize WinSock
	int iRes = WSAStartup(MAKEWORD(2, 2), &wasData);
	if (iRes != 0)
	{
			printf("WSAStartup failed: %d\n" , iRes);
			return 1;
	} 
```

调用WSAStartup函数来启动WS2_32.dll的使用。

WSADATA结构包含关于Windows套接字实现的信息。WSAStartup的MAKEWORD(2,2)参数在系统上请求Winsock的2.2版本，并将传递的版本设置为调用者可以使用的Windows套接字支持的最高版本。

## 2. 创建server

参考[1.1](#11_ServerClient_8)中的步骤，在[1.3](#13_Winsock_73)中已经说明了如何初始化Winsock，下面应该是创建server socket

### 2.1 创建server socket

1.  使用[getaddrinfo()](https://docs.microsoft.com/en-us/windows/win32/api/ws2tcpip/nf-ws2tcpip-getaddrinfo)确定sockaddr结构体值，getaddrinfo中使用[addrinfo](https://docs.microsoft.com/en-us/windows/win32/api/ws2def/ns-ws2def-addrinfoa)结构体。

使用的信息包含以下内容：

字段|作用|
---|---
AF_INET|指定IPv4地址族
SOCK_STREAM|指定一个流套接字
IPPROTO_TCP|指定TCP协议
AI_PASSIVE|AI_PASSIVE标志表示调用者打算在调用bind函数时使用返回的套接字地址结构。当AI_PASSIVE标志被设置并且getaddrinfo函数的nodename参数是一个空指针时，套接字地址结构的IP地址部分被设置为IPv4地址INADDR_ANY或IPv6地址IN6ADDR_ANY_INIT。

代码如下：
```cpp
   #define DEFAULT_PORT "27015"

// 2. create server socket
	addrinfo* result = NULL, * ptr = NULL, hints;

	ZeroMemory(&hints, sizeof(hints));
	hints.ai_family = AF_INET;
	hints.ai_socktype = SOCK_STREAM;
	hints.ai_protocol = IPPROTO_TCP;
	hints.ai_flags = AI_PASSIVE;

	// Resolve the local address and port to be used by the server
	iRes = getaddrinfo(NULL, DEFAULT_PORT, &hints, &result);
	if (iRes != 0)
	{
		printf("getaddrinfo failed:%d\n",iRes);
		WSACleanup();
		return 1;
	}
```

<div id="refer-anchor-212"></div>

2. 创建SOCKET对象ListenSocket ，用来监听客户端连接请求。

```cpp
SOCKET ListenSocket = INVALID_SOCKET;
```

<div id="refer-anchor-213"></div>

3. 调用socket函数，返回值赋给ListenSocket 。

对于server，使用getaddrinfo返回的第一个IP地址，该IP地址与在提示参数中指定的地址家族、套接字类型和协议相匹配

如果想监听IPv6，ai_family = AF_INET6；

如果想同时监听IPv4和IPv6，必须创建两个监听套接字，一个监听IPv6，一个监听IPv4。应用程序必须分别处理这两个套接字。

```cpp
ListenSocket = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
```

<div id="refer-anchor-214"></div>

4. 检查错误，确保socket是一个有效的套接字

```cpp
	// Check for errors to ensure that the socket is valid socket
	if (ListenSocket == INVALID_SOCKET)
	{
		cout << "Error at socket():"<< WSAGetLastError() << endl;
		freeaddrinfo(result);
		WSACleanup();
		return 1;
	}
```

### 2.2 绑定socket
server如果要接收client连接请求，需要绑定一个网络地址。下面阐述如果绑定一个创建了IP地址和端口的socket。client使用IP地址和端口连接主机。

1. bind并检查错误

sockaddr结构保存有关地址家族、IP地址和端口号的信息。

调用`bind()`，传递创建的socket和getaddrinfo函数返回的sockaddr结构作为参数。检查一般性错误。

```cpp
	// 3. Bind socket
	// Setup the TCP listening socket
	iRes = bind(ListenSocket, result->ai_addr, (int)result->ai_addrlen);
	if (iRes == SOCKET_ERROR)
	{
		cout << "bind failed with error: " << WSAGetLastError() << endl;
		freeaddrinfo(result);
		closesocket(ListenSocket);
		WSACleanup();
		return 1;
	}
```
2. 释放存放地址信息的内存空间

一旦绑定完成，getaddrinfo获取的地址信息就不在需要了，使用freeaddrinfo释放分配的内存。

```cpp
	// free memory allocated by getaddrinfo() for address information
	freeaddrinfo(result);
```

### 2.3 监听
socket绑定IP地址和端口后，需要监听该IP和端口发送的连接请求。

调用`listen()`将创建的socket和待定的值(待定连接队列的最大长度)作为参数传递。在本例中，backlog参数被设置为SOMAXCONN。此值是一个特殊常量，指示此套接字的Winsock提供程序允许队列中挂起连接的最大合理数量。检查返回值是否有一般错误。

### 2.4 接受连接请求

监听时若收到连接请求，需处理该请求。

1. 创建临时SOCKET对象ClientSocket接受client的连接

```cpp
// 5. Accepting a Connetion
// Create temporary ClientSocket for accepting connetions from clients
SOCKET ClientSocket = INVALID_SOCKET;
```

2. 通常server要监听多个客户端的连接请求。对一个高性能的server来说，需要使用多线程处理多客户端请求。

Winsock有多种处理多客户端连接请求的技术。一种编程技术是创建一个连续循环，使用`listen()`检查连接请求(参见[2.3](#23__195))。如果出现连接请求，应用程序将调用`accept、AcceptEx或WSAAccept`函数，并将工作传递给另一个线程来处理请求。还可以使用其他几种编程技术。

>**Note**
>这个基本示例非常简单，并且不使用多线程。该示例还只侦听和接受单个连接。

```cpp
    // Accept a client socket
	ClientSocket = accept(ListenSocket, NULL, NULL);
	if (ClientSocket == INVALID_SOCKET)
	{
		cout << "accept failed: " << WSAGetLastError() << endl;
		closesocket(ListenSocket);
		WSACleanup();
		return 1;
	}
```

### 2.5 接收好发送数据

使用`recv()`和`send()`接收和发送消息
```cpp
#define DEFAULT_BUFLEN 512

	// 6. Receiving and Sending Data on the Server
	char recvbuf[DEFAULT_BUFLEN];
	int iSendRes;
	int recvbuflen = DEFAULT_BUFLEN;

	iRes = 1;
	// Receive until the peer shuts down the connection
	while (iRes > 0)
	{
		iRes = recv(ClientSocket, recvbuf, recvbuflen, 0);
		if (iRes > 0)
		{
			cout << "Bytes received: " << iRes << endl;

			// Echo the buffr back to the sender
			iSendRes = send(ClientSocket, recvbuf, iRes, 0);
			if (iSendRes == SOCKET_ERROR)
			{
				cout << "send failed: " << WSAGetLastError() << endl;
				closesocket(ClientSocket);
				WSACleanup();
				return 1;
			}
			cout << "Bytes sent: " << iSendRes << endl;
		}
		else
		{
			cout <<"recv failed: " << WSAGetLastError() << endl;
			closesocket(ClientSocket);
			WSACleanup();
			return 1;
		}
	}
```

### 2.6 断开连接

1. 当server完成向client发送数据时，可以调用`shutdown()`，指定`SD_SEND`来关闭套接字的发送端。这允许客户端释放此套接字的一些资源。服务器应用程序仍然可以接收套接字上的数据。
```cpp
	// 7.Disconneting the Server
	// shutdown the send half of the connetiong since no more data will be sent
	iRes = shutdown(ClientSocket, SD_SEND);
	if (iRes == SOCKET_ERROR)
	{
		cout << "shutdown failed: " << WSAGetLastError() << endl;
		closesocket(ClientSocket);
		WSACleanup();
		return 1;
	}
```

2. 当客户端应用程序完成接收数据时，将调用`closesocket()`来关闭套接字。

当客户端应用程序使用Windows套接字DLL完成时，WSACleanup函数被调用来释放资源。

```cpp
	// cleanup
	closesocket(ClientSocket);
	WSACleanup();
	return 0;
```

## 3. 创建client
与第2节大体相似。

### 3.1 创建client socket

1. 对于这个应用程序，Internet地址族是未指定的`AF_UNSPEC`，因此可以返回IPv6或IPv4地址。其余与[2.1](#21_server_socket_99)的1基本相同

```cpp
struct addrinfo *result = NULL,
                *ptr = NULL,
                hints;

ZeroMemory( &hints, sizeof(hints) );
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;
hints.ai_protocol = IPPROTO_TCP;
```

2. 与[2.1](#21_server_socket_99)的1不同的是，请求在命令行中传递的服务器名称的IP地址。

```cpp
#define DEFAULT_PORT "27015"

// Resolve the server address and port
iResult = getaddrinfo(argv[1], DEFAULT_PORT, &hints, &result);
if (iResult != 0) {
    printf("getaddrinfo failed: %d\n", iResult);
    WSACleanup();
    return 1;
}
```

3. 同[2.1](#21_server_socket_99)的[2](#refer-anchor-212)

```cpp
SOCKET ConnectSocket = INVALID_SOCKET;
```

4. 同[2.1](#21_server_socket_99)的[3](#refer-anchor-213)

```cpp
// Attempt to connect to the first address returned by
// the call to getaddrinfo
ptr=result;

// Create a SOCKET for connecting to server
ConnectSocket = socket(ptr->ai_family, ptr->ai_socktype, 
    ptr->ai_protocol);
```

5. 同[2.1](#21_server_socket_99)的[4](#refer-anchor-214)

```cpp
if (ConnectSocket == INVALID_SOCKET) {
    printf("Error at socket(): %ld\n", WSAGetLastError());
    freeaddrinfo(result);
    WSACleanup();
    return 1;
}
```

### 3.2 连接server

客户端想要通信，需要连接server。

调用`connect()`，设置参数为创建的socket和sockaddr结构，并检查错误。

```cpp
// Connect to server.
iResult = connect( ConnectSocket, ptr->ai_addr, (int)ptr->ai_addrlen);
if (iResult == SOCKET_ERROR) {
    closesocket(ConnectSocket);
    ConnectSocket = INVALID_SOCKET;
}

// Should really try the next address returned by getaddrinfo
// if the connect call failed
// But for this simple example we just free the resources
// returned by getaddrinfo and print an error message

freeaddrinfo(result);

if (ConnectSocket == INVALID_SOCKET) {
    printf("Unable to connect to server!\n");
    WSACleanup();
    return 1;
}
```

在本例中，`getaddrinfo()`返回的第一个IP地址用于指定传递给连接的`sockaddr`结构。如果对第一个IP地址的连接调用失败，那么尝试从`getaddrinfo()`返回的链表中的下一个`addrinfo`结构。

sockaddr结构中指定的信息包括:

- 客户机将尝试连接到的服务器的IP地址。
- 客户机将连接到的服务器端口号。当
- 客户端调用`getaddrinfo()`时，该端口被指定为端口27015。

### 3.3 发送接收数据

```cpp#define DEFAULT_BUFLEN 512
int recvbuflen = DEFAULT_BUFLEN;

const char *sendbuf = "this is a test";
char recvbuf[DEFAULT_BUFLEN];

int iResult;

// Send an initial buffer
iResult = send(ConnectSocket, sendbuf, (int) strlen(sendbuf), 0);
if (iResult == SOCKET_ERROR) {
    printf("send failed: %d\n", WSAGetLastError());
    closesocket(ConnectSocket);
    WSACleanup();
    return 1;
}

printf("Bytes Sent: %ld\n", iResult);

// shutdown the connection for sending since no more data will be sent
// the client can still use the ConnectSocket for receiving data
iResult = shutdown(ConnectSocket, SD_SEND);
if (iResult == SOCKET_ERROR) {
    printf("shutdown failed: %d\n", WSAGetLastError());
    closesocket(ConnectSocket);
    WSACleanup();
    return 1;
}

// Receive data until the server closes the connection
do {
    iResult = recv(ConnectSocket, recvbuf, recvbuflen, 0);
    if (iResult > 0)
        printf("Bytes received: %d\n", iResult);
    else if (iResult == 0)
        printf("Connection closed\n");
    else
        printf("recv failed: %d\n", WSAGetLastError());
} while (iResult > 0);
```

### 3.4 断开连接
同[2.6](#26__280)

## 4. 完整应用代码
### 4.1 Server
```cpp
// FileName: server.cpp
// Description: Create server socket application
// Author: Jiejing.Ma
// Update: 2020/12/11

#undef UNICODE

#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif // !WIN32_LEAN_AND_MEAN

#include <Windows.h>
#include <WinSock2.h>  // socket
#include <WS2tcpip.h>  // TCP/IP
#include <iphlpapi.h>  // ip helper APIs
#include <stdlib.h>
#include <iostream>

// Link to Ws2_32.lib
#pragma comment(lib,"Ws2_32.lib")

#define DEFAULT_BUFLEN 512
#define DEFAULT_PORT "27015"

using namespace std;

int main()
{
	WSADATA wasData;
	int iRes;

	// Create a SOCKET object to listen for client connections
	SOCKET ListenSocket = INVALID_SOCKET;
	// Create temporary ClientSocket for accepting connetions from clients
	SOCKET ClientSocket = INVALID_SOCKET;

	addrinfo* result = NULL;
	addrinfo hints;

	char recvbuf[DEFAULT_BUFLEN];
	int iSendRes;
	int recvbuflen = DEFAULT_BUFLEN;

	// 1. Initialize WinSock
	iRes = WSAStartup(MAKEWORD(2, 2), &wasData);
	if (iRes != 0)
	{
		cout << "WSAStartup failed: " << iRes << endl;
		return 1;
	}

	// 2. Create server socket
	ZeroMemory(&hints, sizeof(hints));
	hints.ai_family = AF_INET;
	hints.ai_socktype = SOCK_STREAM;
	hints.ai_protocol = IPPROTO_TCP;
	hints.ai_flags = AI_PASSIVE;

	// Resolve the local address and port to be used by the server
	iRes = getaddrinfo(NULL, DEFAULT_PORT, &hints, &result);
	if (iRes != 0)
	{
		cout << "getaddrinfo failed: " << iRes << endl;
		WSACleanup();
		return 1;
	}

	// Create a SOCKET for connecting to server
	ListenSocket = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
	// Check for errors to ensure that the socket is valid socket
	if (ListenSocket == INVALID_SOCKET)
	{
		cout << "Error at socket():"<< WSAGetLastError() << endl;
		freeaddrinfo(result);
		WSACleanup();
		return 1;
	}

	// 3. Bind socket
	// Setup the TCP listening socket
	iRes = bind(ListenSocket, result->ai_addr, (int)result->ai_addrlen);
	if (iRes == SOCKET_ERROR)
	{
		cout << "bind failed with error: " << WSAGetLastError() << endl;
		freeaddrinfo(result);
		closesocket(ListenSocket);
		WSACleanup();
		return 1;
	}
	// free memory allocated by getaddrinfo() for address information
	freeaddrinfo(result);

	// 4. Listening on a Socket
	if (listen(ListenSocket, SOMAXCONN) == SOCKET_ERROR)
	{
		cout << "Listen failed with error: " << WSAGetLastError() << endl;
		closesocket(ListenSocket);
		WSACleanup();
		return 1;
	}

	// 5. Accepting a Connetion
	// Accept a client socket
	ClientSocket = accept(ListenSocket, NULL, NULL);
	if (ClientSocket == INVALID_SOCKET)
	{
		cout << "accept failed: " << WSAGetLastError() << endl;
		closesocket(ListenSocket);
		WSACleanup();
		return 1;
	}

	// No longer need server socket
	closesocket(ListenSocket);

	// 6. Receiving and Sending Data on the Server
	iRes = 1;
	// Receive until the peer shuts down the connection
	while (iRes > 0)
	{
		iRes = recv(ClientSocket, recvbuf, recvbuflen, 0);
		if (iRes > 0)
		{
			cout << "Bytes received: " << iRes << endl;

			// Echo the buffr back to the sender
			iSendRes = send(ClientSocket, recvbuf, iRes, 0);
			if (iSendRes == SOCKET_ERROR)
			{
				cout << "send failed: " << WSAGetLastError() << endl;
				closesocket(ClientSocket);
				WSACleanup();
				return 1;
			}
			cout << "Bytes sent: " << iSendRes << endl;
		}
		else
		{
			cout <<"recv failed: " << WSAGetLastError() << endl;
			closesocket(ClientSocket);
			WSACleanup();
			return 1;
		}
	}

	// 7.Disconneting the Server
	// shutdown the send half of the connetiong since no more data will be sent
	iRes = shutdown(ClientSocket, SD_SEND);
	if (iRes == SOCKET_ERROR)
	{
		cout << "shutdown failed: " << WSAGetLastError() << endl;
		closesocket(ClientSocket);
		WSACleanup();
		return 1;
	}

	// cleanup
	closesocket(ClientSocket);
	WSACleanup();

	return 0;
}
```

### 4.2 Client

```cpp
// FileName: client.cpp
// Description: Create client socket application
// Author: Jiejing.Ma
// Update: 2020/12/11

#undef UNICODE

#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif // !WIN32_LEAN_AND_MEAN

#include <Windows.h>
#include <WinSock2.h>
#include <WS2tcpip.h>
#include <iphlpapi.h>
#include <iostream>

// Need to link with Ws2_32.li, Mswsock.lib, Advapi32.lib
#pragma comment(lib,"Ws2_32.lib")
#pragma comment(lib,"Mswsock.lib")
#pragma comment(lib,"Advapi32.lib")

#define DEFAULT_BUFLEN 512
#define DEFAULT_PORT "27015"

using namespace std;

int main(int argc,char ** argv)
{
	WSADATA wsaData;
	int iRes;

	SOCKET ConnectSocket  = INVALID_SOCKET;

	addrinfo hints;
	addrinfo* result = NULL, *ptr=NULL;

	const char *sendbuf="hello";
	char recvbuf[DEFAULT_BUFLEN];
	int recvbuflen = DEFAULT_BUFLEN;

	// 1. Initialize WinSock
	iRes = WSAStartup(MAKEWORD(2, 2), &wsaData);
	if (iRes != 0)
	{
		cout << "WSAStartup failed: " << iRes << endl;
		return 1;
	}

	// 2. Create socket
	ZeroMemory(&hints, sizeof(hints));
	hints.ai_family = AF_UNSPEC;
	hints.ai_protocol = IPPROTO_TCP;
	hints.ai_socktype = SOCK_STREAM;

	// Resolve the server address and port
	iRes = getaddrinfo(argv[1], DEFAULT_PORT, &hints, &result);
	if (iRes != 0)
	{
		cout << "getaddrinfo failed:" << iRes << endl;
		WSACleanup();
		return 1;
	}

	// Attempt to connect to an address until one succeeds
	for (ptr = result; ptr != NULL; ptr = ptr->ai_next)
	{
		// Create a SOCKET for connecting to server	
		ConnectSocket  = socket(ptr->ai_family, ptr->ai_socktype, ptr->ai_protocol);
		if (ConnectSocket  == INVALID_SOCKET)
		{
			cout << "Error at socket():" << WSAGetLastError() << endl;
			WSACleanup();
			return 1;
		}

		// 3.Connect to Server
		iRes = connect(ConnectSocket , ptr->ai_addr, (int)ptr->ai_addrlen);
		if (iRes == SOCKET_ERROR)
		{
			closesocket(ConnectSocket );
			ConnectSocket  = INVALID_SOCKET;
			continue;
		}
		break;
	}

	freeaddrinfo(result);

	if (ConnectSocket  == INVALID_SOCKET)
	{
		cout << "Unable to connect to server!" << endl;
		WSACleanup();
		return 1;
	}
	// 4. Send and Receive data
	// Send an initial buffer
	iRes = send(ConnectSocket , sendbuf, (int)strlen(sendbuf), 0);
	if (iRes == SOCKET_ERROR)
	{
		cout << "send faild: " << WSAGetLastError() << endl;
		closesocket(ConnectSocket );
		WSACleanup();
		return 1;
	}
	cout << "Bytes sent: " << iRes << endl;

	// shutdown the connection for sending since no more data will be sent
	// the client can still use the ConnectSocket for receiving data
	iRes = shutdown(ConnectSocket , SD_SEND);
	if (iRes == SOCKET_ERROR)
	{
		cout << "shutdown failed: " << WSAGetLastError() << endl;
		closesocket(ConnectSocket );
		WSACleanup();
		return 1;
	}

	iRes = 1;
	while (iRes>0)
	{
		iRes = recv(ConnectSocket , recvbuf, recvbuflen, 0);
		if (iRes > 0)
			printf("Bytes received: %d\n", iRes);
		else if (iRes == 0)
			printf("Connection closed\n");
		else
			printf("recv failed: %d\n", WSAGetLastError());
	}

	// 5. Disconnect
	// cleanup
	closesocket(ConnectSocket );
	WSACleanup();

	return 0;
}
```