---
layout:     post
title:      "socket编程基础"
subtitle:   "Basic socket programming"
date:       2022-06-11 19:16:00
author:     "Ethan"
catalog: false
header-style: text
tags:
    - 编程基础
    - 网络编程
---
>Everything is a file including Internet, which shows the power of abstraction

最近在找实习，网上冲浪时发现很多面试官喜欢问网络编程和多线程，而我两个其实都不怎么懂，哎，书到用时方恨少，今天先写一篇介绍socket的。

我们都知道传输层两大协议TCP和UDP实现了两个主机之间的通信，socket更像是个接口，编程时可以选择使用TCP还是UDP协议，它的功能是完成端对端的传输，一个端（endpoint）是由IP地址加端口号唯一指定的，因此为了完成两个主机之间的通信，需要两个socket，分别是服务端和客户端。

值得注意的是，socket其实是个文件描述符，因此socket编程时接收/发送远程主机的数据仿佛就是在读取/写入本地文件一样，我也对**一切都是文件**这个概念有了更深的体会。说句题外话，还有啥意想不到的东西是文件呢？其实平时用的键盘对应的是标准输入文件stdin，看的显示器对应的是标准输出文件stdout。
# 整体流程

## server
1. 创建socket
2. 绑定端口-bind
3. 监听端口-listen
4. 接受请求-accept
5. 进行通信-write&read
6. 释放资源-close

## client
1. 创建socket
2. 请求连接-connect
3. 进行通信-write&read
4. 释放资源-close

# 具体流程--server

## 创建socket
`int socket(int af, int type, int protocol)`  
功能：创建socket并确定相关属性  
返回值是新创建的socket的文件描述符，一共有三个参数，分别是：  
- af：地址族，常用的有AF_INET和AF_INET6，分别表示IPv4和IPv6
- type：数据传输方式，常用的有SOCK_STREAM和SOCK_DGRAM，分别表示面向连接的和无连接的数据传输
- protocol：传输协议，常用的有IPPROTO_TCP和IPPROTO_UDP，分别表示TCP和UDP协议

一般情况下，只需要前两个参数就能确定使用什么协议了，所以第三个参数置为0就行，系统自动推算应该选取什么协议。

## 绑定端口
`int bind(int sock, struct sockaddr *addr, socklen_t addrlen)`  
功能：将创建的socket和指定的IP地址和端口号绑定  
返回值如果是0表示成功，-1表示失败，一共有三个参数，分别是：  
- sock：新创建的socket
- addr：一个指向sockaddr结构体的指针，包含了ip地址，端口号等信息
- addrlen：addr指向的结构体的大小

注意到第二个参数是指针，sockaddr结构体的定义很简单，两个成员变量，一共16个字节，但不同的协议会有不同的属性，也就是不同的协议对这16个字节的使用方式是不同的，因此我们通常是先定义一个更为具体的结构体，但它同样是16字节，比如使用IPv4地址时我们会使用sockaddr_in结构体，对它进行赋值后，再转换为sockaddr结构体，而IPv6地址则用sockaddr_in6结构体。这样做的好处是，不同类型的IP地址都能用同一个bind函数。  
sockaddr结构体定义如下：  
```
struct sockaddr{
    sa_family_t  sin_family;   //地址族（Address Family），也就是地址类型
    char         sa_data[14];  //IP地址和端口号
};
```
sockaddr_in结构体（用于保存IPv4地址信息）定义如下：
```
struct sockaddr_in{
    sa_family_t     sin_family;   //地址族（Address Family），也就是地址类型
    uint16_t        sin_port;     //16位的端口号
    struct in_addr  sin_addr;     //32位IP地址
    char            sin_zero[8];  //不使用，一般用0填充
};
```
sockaddr_in6结构体（用于保存IPv6地址信息）定义如下：
```
struct sockaddr_in6 { 
    sa_family_t sin6_family;  //(2)地址类型，取值为AF_INET6
    in_port_t sin6_port;  //(2)16位端口号
    uint32_t sin6_flowinfo;  //(4)IPv6流信息
    struct in6_addr sin6_addr;  //(4)具体的IPv6地址
    uint32_t sin6_scope_id;  //(4)接口范围ID
};
```

## 监听端口
`int listen(int sock, int backlog)`  
功能：让创建的socket监听相应端口是否有请求  
返回值如果是0表示成功，-1表示失败，一共有两个参数，分别是：
- sock：之前创建的socket
- backlog：请求队列的最大长度，也就是最多能容纳多少个客户端的请求，但请求只能一个一个依次处理

## 接受请求
`int accept(int sock, struct sockaddr *addr, socklen_t *addrlen)`  
功能：同意与一个客户端建立连接  
返回值是一个新的socket，这个socket用来之后和客户端的通信，那原来的socket干啥呢？没错，继续监听是否有别的客户端请求，该函数一共有三个参数，分别是：
- sock：之前创建的socket
- addr：一个指向sockaddr的结构体指针，同上，使用时要强制类型转换
- addrlen：addr指向的结构体的大小

第二个参数记录了客户端的IP地址和端口信息，但其实编程时我们并不知道客户端的IP地址和端口信息啊，没错，这个参数是输入给accept函数，在函数体内完成赋值的，accept函数从请求队列里选取第一个客户端请求，把它的IP地址和端口信息赋值给这个指针指向的结构体，然后返回一个新创建的socket，这个socket用来之后和客户端的通信。  
请注意，listen后面的代码会继续执行，但accpet会检查请求队列是否为空，如果为空则一直等待，即阻塞程序的执行，如果不为空则选取队列的第一个连接请求进行建立连接

## 进行通信
因为socket就是个文件描述符，所以通信过程就跟写入/读取文件一样

### 发送
`ssize_t write(int fd, const void *buf, size_t nbytes)`  
功能：将缓冲区buf的前nbytes个字节写入fd指示的文件  
返回值是成功写入的字节数，如果失败则返回-1，一共有三个参数，分别是：
- fd：accept函数返回的新socket
- buf：存储发送数据的缓冲区
- nbytes：准备发送（写入）的字节数

### 接收
`ssize_t read(int fd, void *buf, size_t nbytes)`  
功能：从fd指示的文件读取前nbytes字节到缓冲区buf里，注意：nbytes是这次读取操作能读到的最多的字节数，不一定每次都能读到这么多  
返回值是成功读取到的字节数（如果遇到文件末尾EOF则返回0），如果失败则返回-1，一共有三个参数，分别是：
- fd：accept函数返回的新socket
- buf：存储接收信息流的缓冲区
- nbytes：准备接收（读取）的字节数

### socket缓冲区
socket被创建后，会分配两个缓冲区，分别是接收缓冲区和发送缓冲区，**注意**：这两个缓冲区对程序员是不可见的，和上面write/read函数声明里的缓冲区buf完全不是一回事。  
严格来讲，write函数不是直接将数据写入另一个主机，而是仅仅将数据写入发送缓冲区，然后交由TCP协议去发送给目标主机，如果正在发送数据或者发送缓冲区满了，那么write函数是不能写入的，也就是会被阻塞；同理，read函数也不是直接读取另一个主机的数据，而是读取接收缓冲区的数据，接收缓冲区的数据是由TCP协议根据接受到的数据自动填充的。  
发送缓冲区发送出去的数据，只有在收到了相应的ACK后才会被清除，为后续待发的数据腾出空间，如果没有收到ACK，意味着可能存在丢包现象，需要重发。接受端则是在收到数据后自动发送ACK确认报文，但如果接收端的应用程序一直不去读取接收缓冲区，那么久而久之接收缓冲区就会满，那么接收端就不会再接收新的数据了，也就不存在回送ACK报文了，因此发送端一直收不到新的ACK，数据一直得不到清除，发送缓冲区也会满，write函数就会被阻塞。

## 释放资源
`int close (int __fd)`  
功能：关闭socket，不再与另一主机通信（包括接收和发送）  
`int shutdown(int sock, int howto)`  
功能：关闭连接，一共有3种（由参数howto指定），分别是：
- 关闭接收
- 关闭发送
- 同时关闭接收和发送

# 具体流程--client

## 创建socket
同服务端

## 请求连接
`int connect(int sock, struct sockaddr *serv_addr, socklen_t addrlen)`  
功能：请求和指定服务端建立连接  
返回值如果是0表示成功，-1表示失败，一共有三个参数，分别是：
- sock：新创建的socket
- serv_addr：指示了服务器的IP地址和端口号，同上，需要类型转换，这是为了能够同时支持IPv4和IPv6地址
- addrlen：serv_addr指示的结构体的大小

## 进行通信
同上，包含write和read函数

## 释放资源
同上，包含close和shutdown两种

# 代码示例
以下示例实现了一个回声服务器，客户端输入除"quit"之外的任何字符串，服务端就返回相同的字符串，这个过程可以重复进行，输入"quit"则退出程序。

## server.cpp
```
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>
#include<netinet/in.h>
#define PORT 8080
#define BUFSIZE 256

void error(const char *msg){
    perror(msg);
    exit(1);
}

int main(){
    //create socket
    int sock=socket(AF_INET,SOCK_STREAM,0);
    if(sock<0){
        error("ERROR on creating socket");
    }

    //bind
    struct sockaddr_in serv_addr;
    memset(&serv_addr,0,sizeof(serv_addr));//fill zero
    serv_addr.sin_family=AF_INET;//IPv4
    serv_addr.sin_addr.s_addr=INADDR_ANY;//filled with current host IP address automatically
    serv_addr.sin_port=htons(PORT);//port number
    if(bind(sock,(struct sockaddr*)&serv_addr,sizeof(serv_addr))<0){
        error("ERROR on binding");
    }

    //listen
    listen(sock,5);//5 is max queue length

    char buf[BUFSIZE];
    struct sockaddr_in cli_addr;
    socklen_t cli_addr_size=sizeof(cli_addr);

    //accept
    int newsock=accept(sock,(struct sockaddr*)&cli_addr,&cli_addr_size);
    //newsock is used for communication with client while sock is still used for accepting incoming connections
    printf("server: connected with %s port: %d\n",inet_ntoa(cli_addr.sin_addr),ntohs(cli_addr.sin_port));

    while(true){
        //read message
        int nRead=read(newsock,buf,BUFSIZE);
        printf("receive %d bytes from client\n",nRead);
        if(strcmp(buf,"quit")==0){
            printf("Disconnected\n");
            break;
        }

        //send message
        int nWrite=write(newsock,buf,nRead);
        printf("send %d bytes to client\n",nWrite);
        
        //clear buffer
        memset(buf,0,BUFSIZE);
    }
    //close socket
    close(newsock);
    close(sock);
}
```

# client.cpp
```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define PORT 8080
#define BUFSIZE 256

void error(const char *msg){
    perror(msg);
    exit(1);
}

int main(){
    //server addr
    struct sockaddr_in serv_addr;
    memset(&serv_addr,0,sizeof(serv_addr));
    serv_addr.sin_family=AF_INET;
    serv_addr.sin_addr.s_addr=INADDR_ANY;
    serv_addr.sin_port=htons(PORT);
    
    //create socket
    int sock=socket(AF_INET,SOCK_STREAM,0);
    if(sock<0){
        error("ERROR on creating socket");
    }

    //connect
    if(connect(sock,(struct sockaddr*)&serv_addr,sizeof(serv_addr))<0){
        error("ERROR on connecting");
    }

    char bufSend[BUFSIZE],bufRecv[BUFSIZE];
    while(true){
        //send message
        printf("Input a string:");
        scanf("%[^\n]",bufSend);
        getchar();
        int nWrite=write(sock,bufSend,strlen(bufSend));
        printf("send %d bytes to server\n",nWrite);

        if(strcmp(bufSend,"quit")==0){
            printf("Disconnected\n");
            break;
        }

        //recv message
        int nRead=read(sock,bufRecv,BUFSIZE);
        printf("Message from server: %s, %d bytes\n",bufRecv,nRead);

        //clear buffer
        memset(bufSend,0,BUFSIZE);
        memset(bufRecv,0,BUFSIZE);
    }  
    //close socket
    close(sock);
}
```

# References
- https://www.bogotobogo.com/cplusplus/sockets_server_client.php
- http://c.biancheng.net/cpp/html/3029.html
