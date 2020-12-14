
---
title: 在C++与python间传视频帧
date: 2020-11-30 21:37:33
author: 马捷径
img: https://img-blog.csdnimg.cn/20201130204440444.png
top: true
cover: true
coverImg: https://img-blog.csdnimg.cn/20201130204440444.png
toc: true
mathjax: true
summary: 本案例旨在实现跨语言（C++和python间）视频的实时通信。这一工作内容在实际工程中很常见。由于python语言支持很多第三方库，对于开发深度学习项目很方便，验真算法速度快，很多开源算法也大多基于python实现。这时可能就会出现C++的代码借助python语言做一些图像处理（包括目标检测、姿态估计、目标跟踪等任务）的需求。
keywords: 
  - 通信 
  - 共享内存
  - socket
  - C++
  - python
categories: Communication
tags:
  - 共享内存
  - socket
---

# 引言
本案例旨在实现跨语言（C++和python间）视频的实时通信。这一工作内容在实际工程中很常见。由于python语言支持很多第三方库，对于开发深度学习项目很方便，验真算法速度快，很多开源算法也大多基于python实现。这时可能就会出现C++的代码借助python语言做一些图像处理（包括目标检测、姿态估计、目标跟踪等任务）的需求。

平台环境：
- Win10
- VS2019
- OpenCV

进程间通信方式：共享内存

# 1.进程间通信

进程间通信方式有很多种。工程上最常用的是**共享内存**和**socket机制**。前者效率高，基本思想就是开辟一块公共的内存空间，供两个或多个进程之间使用。为了标识这个公共空间，要给它起个名。但是共享内存的方式不支持多平台。而socket刚好就是支持多平台间进程通信的方式。当然这种方式也会慢一些。

在本案例中，分别尝试了两种方式。虽然最终共享内存的方式写内存帧率只达到15fps左右，但是要比socket快了近20倍（大概0.5-1fps左右）。下面将介绍这两种机制的具体实现过程。

# 2.基于共享内存的视频传输
## 2.1 C++之间的通信
### 2.1.1 接口函数
首先验证C++之间能通信。这里使用的是[CreateFileMapping](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga)和[MapViewOfFile](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile)进行共享内存的创建和映射。

其中[CreateFileMapping](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga)的接口为，参数含义详解请点击链接。
```cpp
HANDLE CreateFileMapping(
  HANDLE                hFile,
  LPSECURITY_ATTRIBUTES lpFileMappingAttributes,
  DWORD                 flProtect,
  DWORD                 dwMaximumSizeHigh,
  DWORD                 dwMaximumSizeLow,
  LPCSTR                lpName
);
```

[MapViewOfFile](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile)的接口为
```cpp
LPVOID MapViewOfFile(
  HANDLE hFileMappingObject,
  DWORD  dwDesiredAccess,
  DWORD  dwFileOffsetHigh,
  DWORD  dwFileOffsetLow,
  SIZE_T dwNumberOfBytesToMap
);
```
### 2.1.2 创建数据格式和共享内存信息
首先需要一个图像的头部
```cpp
typedef struct {
	int width;
	int height;
	int type;
}ImgInf;       //图像信息
```
由于sizeof(int)=4，所以这里ImgInf结构体大小为12B。在进行共享内存映射时，我们需要这个大小去做偏移量，找到图像数据。

接下来要定义图像的数据信息
```cpp
#define FRAME_NUMBER         1               // 图像路数
#define FRAME_W              1920
#define FRAME_H              1080
#define FRAME_W_H            FRAME_W*FRAME_H
// 图像分辨率：彩色图（3通道）+图像信息结构体
#define FRAME_SIZE           FRAME_W_H*sizeof(unsigned char)*3+sizeof(ImgInf)

#define MEMORY_SIZE          FRAME_NUMBER*FRAME_SIZE
```
![图像数据空间分配](https://img-blog.csdnimg.cn/20201130102605383.PNG#pic_center)

定义共享内存类SHAREDMEMORY
```cpp
class SHAREDMEMORY
{
public:
	SHAREDMEMORY();
	~SHAREDMEMORY();

	//void SendBox(TrackBox& BOX);
	//void RecBox(TrackBox& BOX);
	//void SendVectorBox(vector<TrackBox>& VTrackBox);
	//void RecieveVectorBox(vector<TrackBox>& VTrackBox);
	void SendMat(cv::Mat img, char indexAddress);
	cv::Mat  ReceiveMat(char indexAddress);
	void SendStr(const char data[]);  
	char* ReceiveStr();

public:
	int state;
private:
	HANDLE hShareMem;                               //共享内存句柄
	TCHAR sShareMemName[30] = TEXT("CppPytonSharedFrame"); // 共享内存名称
	LPCTSTR pBuf;	
};
```
其中SendMat为图像数据发送，ReceiveMat为图像接收。
SendStr为字符串发送，ReceiveStr为字符串接收。

最后的**ShareMemory.h**文件如下：
```cpp
#pragma once
// ShareMemory.h : 此文件包含共享内存数据定义、大小确定、位置分配、信息定义
// Author : Jiejing.Ma
// Update : 2020/11/27
#ifndef ShareMemory_H
#define ShareMemory_H

#include <opencv2/core.hpp>
#include <opencv2/videoio.hpp>
#include <opencv2/highgui.hpp>
#include <opencv2/imgproc.hpp>  // cv::Canny()
#include <opencv2/opencv.hpp>

#include <Windows.h>

//=================================共享内存数据定义=================================
typedef struct {
	int width;
	int height;
	int type;
}ImgInf;       //图像信息
//=================================共享内存大小确定=================================
// 为图像分配空间
#define FRAME_NUMBER         1               // 图像路数
#define FRAME_W              1920
#define FRAME_H              1080
#define FRAME_W_H            FRAME_W*FRAME_H
// 图像分辨率：彩色图（3通道）+图像信息结构体
#define FRAME_SIZE           FRAME_W_H*sizeof(unsigned char)*3+sizeof(ImgInf)

#define MEMORY_SIZE          FRAME_NUMBER*FRAME_SIZE

//=================================共享内存信息定义=================================
#define INITSUCCESS      0
#define CREATEMAPFAILED  1
#define MAPVIEWFAILED    2

class SHAREDMEMORY
{
public:
	SHAREDMEMORY();
	~SHAREDMEMORY();
	void SendMat(cv::Mat img, char indexAddress);
	cv::Mat  ReceiveMat(char indexAddress);
	void SendStr(const char data[]);
	char* ReceiveStr();

public:
	int state;
private:
	HANDLE hShareMem;                               //共享内存句柄
	TCHAR sShareMemName[30] = TEXT("ShareMedia");   // 共享内存名称
	LPCTSTR pBuf;	
};

#endif // !ShareMemory_H
```
对应的**ShareMemory.cpp**文件为类的实现。
```cpp
#pragma once 
// ShareMemory.cpp : 此文件包含信息定义SHAREDMEMOR类的实现
// Author : MJJ
// Update : 2020/11/27
#ifndef ShareMemory_CPP
#define ShareMemory_CPP

#include "ShareMemory.h"
#include <iostream>

using namespace cv;
using namespace std;

/*************************************************************************************
FuncName  :SHAREDMEMORY::~SHAREDMEMORY()
Desc      :构造函数创建共享内存
Input     :None
Output    :None
**************************************************************************************/
SHAREDMEMORY::SHAREDMEMORY() {
	hShareMem = CreateFileMapping(
		INVALID_HANDLE_VALUE,  // use paging file
		NULL,                  //default security
		PAGE_READWRITE,        //read/write access
		0,                     // maximum object size(high-order DWORD)
		MEMORY_SIZE,           //maximum object size(low-order DWORD)
		sShareMemName);        //name of mapping object

	if (hShareMem) {
		//  映射对象视图，得到共享内存指针，设置数据
		pBuf = (LPTSTR)MapViewOfFile(
			hShareMem,           //handle to map object
			FILE_MAP_ALL_ACCESS, // read/write permission
			0,
			0,
			MEMORY_SIZE);
		cout << "memory size:" << MEMORY_SIZE<< endl;

		// 若映射失败退出
		if (pBuf == NULL)
		{
			std::cout << "Could not map view of framebuffer file." << GetLastError() << std::endl;
			CloseHandle(hShareMem);
			state = MAPVIEWFAILED;
		}
	}
	else
	{
		std::cout << "Could not create file mapping object." << GetLastError() << std::endl;
		state = CREATEMAPFAILED;
	}
	state = INITSUCCESS;
}

/*************************************************************************************
FuncName  :SHAREDMEMORY::~SHAREDMEMORY()
Desc      :析构函数释放
Input     :None
Output    :None
**************************************************************************************/
SHAREDMEMORY::~SHAREDMEMORY() {
	std::cout << "unmap shared addr." << std::endl;
	UnmapViewOfFile(pBuf); //释放；
	CloseHandle(hShareMem);
}

/*************************************************************************************
FuncName  :void SHAREDMEMORY::SendMat(cv::Mat img, char indexAddress)
Desc      :发送Mat数据
Input     :
	Mat img               发送图像
	char indexAddress     共享内存中起始位置，若只有一路视频则无偏移
Output    :None
**************************************************************************************/
void SHAREDMEMORY::SendMat(cv::Mat img, char indexAddress) {
	ImgInf img_head;
	img_head.width = img.cols;
	img_head.height = img.rows;
	img_head.type = img.type();

	if (img_head.type == CV_64FC1) {
		memcpy((char*)pBuf + indexAddress, &img_head, sizeof(ImgInf));
		memcpy((char*)pBuf + indexAddress + sizeof(ImgInf),        // Address of dst
			img.data,                                              // Src data
			img.cols * img.rows * img.channels() * sizeof(double)  // size of data
		);
	}
	else
	{
		memcpy((char*)pBuf + indexAddress, &img_head, sizeof(ImgInf));
		memcpy((char*)pBuf + indexAddress + sizeof(ImgInf),        // Address of dst
			img.data,                                              // Src data
			img.cols * img.rows * img.channels()                   // size of data
		);		
	}
	cout << "write shared mem successful." << endl;
}


/*************************************************************************************
FuncName  :cv::Mat SHAREDMEMORY::ReceiveMat(char indexAddress)
Desc      :接收Mat数据
Input     :
	char indexAddress     共享内存中起始位置，若只有一路视频则无偏移
Output    :Mat图像
**************************************************************************************/
cv::Mat SHAREDMEMORY::ReceiveMat(char indexAddress)
{
	ImgInf img_head;
	cv::Mat img;
	memcpy(&img_head, (char*)pBuf + indexAddress, sizeof(ImgInf));
	img.create(img_head.height, img_head.width, img_head.type);
	if (img_head.type == CV_64FC1)
	{
		memcpy(img.data, (char*)pBuf + indexAddress + sizeof(ImgInf), img.cols * img.rows * img.channels() * sizeof(double));
	}
	else
	{
		memcpy(img.data, (char*)pBuf + indexAddress + sizeof(ImgInf), img.cols * img.rows * img.channels());
	}
	return img;
}

/*************************************************************************************
FuncName  :void SHAREDMEMORY::SendStr(cv::Mat img, char indexAddress)
Desc      :发送str数据
Input     :
	Mat img               发送图像
	char indexAddress     共享内存中起始位置，若只有一路视频则无偏移
Output    :None
**************************************************************************************/
void SHAREDMEMORY::SendStr(const char data[]) {
	memcpy((char*)pBuf, data, sizeof(data));
	cout << "write shared mem successful." << endl;
	getchar();
}

/*************************************************************************************
FuncName  :void SHAREDMEMORY::ReceiveStr()
Desc      :接收str数据
Input     :None
Output    :获取的字符串
**************************************************************************************/
char* SHAREDMEMORY::ReceiveStr(){
	char* str= (char*)pBuf;
	cout << "receive is:"<< str << endl;
	return str;
}
#endif // !ShareMemory_CPP
```
### 2.1.3 C++之间共享内存通信
创建一个新的工程，导入上面两个文件，并创建**WriteMem.cpp**文件
```cpp
// WriteMem.cpp : 此文件为写共享内存
// Author : Jiejing.Ma
// Update : 2020/11/27

#include <iostream>
#include "ShareMemory.h"

using namespace std;
using namespace cv;

// 读图片或视频
void send_img(SHAREDMEMORY sharedsend)
{
	int index = 0;
	int64 t0 = cv::getTickCount();;
	int64 t1 = 0;
	string fps = "";
	int nFrames = 0;
	
	cv::Mat frame;

	cout << "Opening video..." << endl;
	VideoCapture cap("test.flv");
	while (cap.isOpened()) 
	{
		cap >> frame;
		if (frame.empty())
		{
			std::cerr << "ERROR: Can't grab video frame." << endl;
			break;
		}
		resize(frame, frame, Size(FRAME_W, FRAME_H));

		nFrames++;
		
		if (!frame.empty()) {
			if (nFrames % 10 == 0)
			{
				const int N = 10;
				int64 t1 = cv::getTickCount();
				fps = " Send FPS:" + to_string((double)getTickFrequency() * N / (t1 - t0)) + "fps";	
				t0 = t1;
			}
			cv::putText(frame, fps, Point(100, 100), cv::FONT_HERSHEY_COMPLEX, 1, cv::Scalar(255, 255, 255),1);
		}
		sharedsend.SendMat(frame, index * FRAME_NUMBER);
		

		if ((waitKey(1) & 0xFF) == 'q') break;
	}
}


int main()
{
	SHAREDMEMORY sharedmem;
	//char str[] = "hello";
	if (sharedmem.state == INITSUCCESS) send_img(sharedmem);
	//if (sharedmem.state == INITSUCCESS) sharedmem.SendStr(str);

	return 0;
}
```

创建一个新的工程，导入上面两个文件，并创建**ReadMem.cpp**文件
```cpp
// ReadMem.cpp : 此文件为读共享内存
// Author : Jiejing.Ma
// Update : 2020/11/27

#include <Windows.h>  
#include <iostream>
#include "ShareMemory.h"
#include <string>

using namespace std;
using namespace cv;

int main(int argc, char** argv)
{
	int index=0;
	SHAREDMEMORY sharemem;
	if (sharemem.state == INITSUCCESS) 
	{
		// read video frame from shared memory.s
		int64 t0 = cv::getTickCount();;
		int64 t1 = 0;
		string fps = "";
		int nFrames = 0;
		namedWindow("ReadMemShow", 0);

		while (true)
		{
			nFrames++;
			Mat frame = sharemem.RecieveMat(index * FRAME_NUMBER);

			if (!frame.empty()) {
				if (nFrames % 10 == 0)
				{
					const int N = 10;
					int64 t1 = cv::getTickCount();
					fps = " Average FPS:" + to_string((double)getTickFrequency() * N / (t1 - t0)) + "fps";
					t0 = t1;
				}
				cv::putText(frame, fps, Point(100, 200), cv::FONT_HERSHEY_COMPLEX, 1, cv::Scalar(100, 200, 200),1);
				imshow("ReadMemShow", frame);				
			}
			if((waitKey(1) & 0xFF) == 'q') break;
		}
		
		//char* str = sharemem.RecieveStr();
	}
	destroyAllWindows();
	return 0;
}
```
同时开启两个工程，则可以接收视频了。

### 2.1.4 C++之间共享内存通信视频测试结果
这里看到，写共享内存速度为15fps，读共享内存速度为65fps（超实时），写速度主要的影响因素与opecv有关。如果优化，还需改视频编解码部分。
![C++和python共享内存传视频测试结果](https://img-blog.csdnimg.cn/20201130204440444.png#pic_center)

## 2.2 C++和python间视频通信
这里以C++作为发送端，python作为接受端。逆向过程还有待测试。网上有教程提到python不能创建共享内存作为发送端，这种说法是错的。本人已测试过，只是发送数据都是字符串型，对于图像数据还有待研究。

### 2.2.1 接口函数

这里主要用到的是**mmap**和**numpy的frombuffer**.

关于mmap，请参考[官网](http://doc.codingdict.com/python_352/library/mmap.html)的接口说明。十分详细，不再赘述。

frombuffer:

```python
numpy.frombuffer(buffer, dtype=float, count=-1, offset=0)
```
>Interpret a buffer as a 1-dimensional array.

>Parameters
bufferbuffer_like
An object that exposes the buffer interface.

>dtypedata-type, optional
Data-type of the returned array; default: float.

>countint, optional
Number of items to read. -1 means all data in the buffer.

>offsetint, optional
Start reading the buffer from this offset (in bytes); default: 0.

参考[官网](https://numpy.org/doc/stable/reference/generated/numpy.frombuffer.html)

### 2.2.1 C++与python之间共享内存通信
前面已经实现C++代码。不需要改动。只需启动写共享内存工程即可。

python代码如下：
```python
import mmap
import os
import cv2
import numpy as np

# -----------------Define info in ShareMemory.h-----------------
IMG_HEAD_OFFSET = 12
# typedef struct {
# 	int width;
# 	int height;
# 	int type;
# }ImgInf;       //图像信息12字节

FRAME_NUMBER = 1
FRAME_W = 1920
FRAME_H = 1080
FRAME_W_H = FRAME_W * FRAME_H
FRAME_SIZE = FRAME_W_H * 3
MEMORY_SIZE = (FRAME_SIZE + IMG_HEAD_OFFSET) * FRAME_NUMBER

sShareMemName = "ShareMedia"

if __name__ == "__main__":
    fpx = mmap.mmap(-1, FRAME_SIZE+IMG_HEAD_OFFSET, sShareMemName)

    # Read img as numpy
    cv2.namedWindow("python_sharedmem_show",0)
    t0 = cv2.getTickCount()
    N = 50
    nFrame = 0
    fps = 0
    while 1:
        nFrame += 1
        img = np.frombuffer(fpx, dtype=np.uint8)
        img = img[IMG_HEAD_OFFSET:FRAME_SIZE+IMG_HEAD_OFFSET]
        img = img.reshape((FRAME_H,FRAME_W,3))

		# Print Average  FPS
        if nFrame % 50 == 0:
            t1 = cv2.getTickCount()
            fps = N*cv2.getTickFrequency() / (t1 - t0)
            t0 = t1
        cv2.putText(img, "Average FPS:" + str(fps) + "fps", (100, 200), cv2.FONT_HERSHEY_COMPLEX, 1, (100, 200, 200), 1)
        cv2.imshow("python_sharedmem_show", img)
        img = None
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cv2.destroyAllWindows()
```

# 3.基于Socket的视频传输
这里是基于Linux开发的Socket通信。而共享内存是基于Windows平台。

## 3.1 cpp端socket
cppsocket.cpp
```cpp
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#include <errno.h>
#include <string>
#include <iostream>
#include <vector>

#include <opencv2/opencv.hpp>

using namespace cv;
using namespace std;

int main(int argc, char **argv)
{
    // 定义socket信息
    char *servInetAddr = "192.168.113.173";
    int servPort = 8081;
    int connfd;
    struct sockaddr_in addr;
    
    // 创建socket
    connfd = socket(AF_INET,SOCK_STREAM, 0);
    if (connfd == -1)
    {
        cout<<"socket创建失败"<<endl;
        exit(-1);
    }

    // 准备通信地址
    addr.sin_family=AF_INET;
    addr.sin_port=htons(servPort);
    addr.sin_addr.s_addr = inet_addr(servInetAddr);

    // bind
    int res = connect(connfd,(struct sockaddr*)&addr,sizeof(addr));
    if(res==-1)
    {
        cout<<"bind连接失败"<<endl;
        exit(-1);
    }
    cout<<"bind连接成功"<<endl;

    // 获取视频帧并发送
    Mat img;
    VideoCapture capture("./test.flv");
    vector<uchar> data_encode;

    while(capture.isOpened()){
        if(!capture.read(img)) break;

        imencode(".jpg",img,data_encode);
        int len_encode = data_encode.size();
        string len = to_string(len_encode);
        int length = len.length(); 
        for (int i=0;i<16-length;i++) len=len+' ';

        // 发送数据
        send(connfd,len.c_str(),strlen(len.c_str()),0);
        char send_char[1];
        for (int i=0;i<len_encode;i++)
        {
            send_char[0]=data_encode[i];
            send(connfd,send_char,1,0);
        }

        // 接收返回信息
        char recvBuf[32] = "";
        if(recv(connfd, recvBuf,32,0)) cout<<recvBuf<<endl;
    }

    close(connfd);
    return 0;

}
```
## 3.2 python端
pysocket.py
```python
import socket
import cv2
import numpy
import time

def recv_size(sock, count):
    buf=b''
    while count:
        newbuf = sock.recv(count)
        if not newbuf: return None
        buf +=newbuf
        count -= len(newbuf)
    return buf


def recv_all(sock, count):
    buf = b''
    while count:
        newbuf = sock.recv(1)
        if not newbuf:return None
        buf +=newbuf
        count -= len(newbuf)
    return buf

# 创建socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# 准备通信地址
address = ('192.168.113.173',8081)
s.bind(address)
s.listen(True)
print('Waiting for images...')
# 接受TCP链接并返回（conn, addr），其中conn是新的套接字对象，可以用来接收和发送数据，addr是链接客户端的地址。
conn, addr = s.accept()
n = 0

while 1:
    n +=1
    length = recv_size(conn,16).decode()
    t0=time.time()
    if isinstance(length,str):  # 若成功接收指定大小信息，进一步接收整张图
        string_data = recv_all(conn,int(length))
        data = numpy.fromstring(string_data,dtype='uint8')
        decimg = cv2.imdecode(data,1)

        cv2.namedWindow('python-recv')
        cv2.imshow('python-recv',decimg)

        if cv2.waitKey(1) & 0xFF == ord('q'):
            print("111111")
            break
        t1 = time.time()
        print('Image recieved successfully!fps:'+str(1/(t1-t0)))
        conn.send('recieved messages!'.encode())
        t0=t1
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

s.close()
cv2.destroyAllWindows()
```
## 3.3 CMakeList
Linux 下编译CPP文件，这里使用CMakeList:
```shell
cmake_minimum_required(VERSION 3.0.0)
project(client VERSION 0.1.0)

include(CTest)
enable_testing()

# find opencv and link
find_package(OpenCV REQUIRED)
message(STATUS "Opencv library status:")
message(STATUS " version:${OpenCV_VERSION}")
message(STATUS " libraries:${OpenCV_LIBS}")
message(STATUS " include path:${OpenCV_INCLUDE_DIRS}")
include_directories(${OpenCV_INCLUDE_DIRS})
link_libraries(${OpenCV_LIBS})

add_executable(client cppsocket.cpp)

set(CMAKE_CXX_FLAGE "${CMAKE_CXX_FLAGE} -g")
```
## 3.4 测试结果
结果就是速度超级慢，大概一秒多一帧。

# 4 结论
C++和python之间通信，可以采用**C++调python**的方式。请参考之前的文章。[Ubuntu下C++调python](https://blog.csdn.net/weixin_38369492/article/details/110090225)
这种方式，从架构的角度来讲，最简单。工程量和已有经验的角度，emm可能坑比较多。速度也应该最快（推测）

也可以使用**进程间通信**。当然这个成本就高了。有两种方式，一是**共享内存**机制，一是**socket通信**。前者更快，但只能在一个平台上。后者慢，可以支持不同电脑间通信。

最后关于基于共享内存方式，影响**速度**的主要是**写共享内存**，而这又与**opencv**读视频有关，与**视频编解码**有关。想要提高写内存速度，需要从底层修改视频编解码。可以参考**UE4**的相关插件解决。