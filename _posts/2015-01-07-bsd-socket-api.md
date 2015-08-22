---
layout: post
title: 'bsd socket api'
date: 2015-01-07 19:44
categories: NetWork
excerpt:
---

* content
{:toc}

* ###int socket ( int family, int type, int protocol)
    * 创建一个socket，返回一个socket描述符`sockFD`，可以理解为一个句柄,是一个small无符号整数,这个句柄可以使用所有其他的socket操作，返回-1时表示跪了   
    * family: 当前协议族只能是`AF_INET`,因为没有支持IPv6，所以没有AF_INET6
    * type: `SOCK_STREAM`是tcp流式，`SOCK_DGRAM`是udp数据包式
    * protocol: `PF_INET`或者是`PF_UNSPEC`,两者都可以，前者只能接受v4的协议，后者通吃。
* ###int socket_bind ( int sockFD, const struct sockaddr *myAddr, int addrLength)
    * 绑定一个socket到本地的端口*and/or*ip地址，不管是ip还是port都可以在库自动选择的情况下得到通配，通配port的话，把address中`sin_port`域设置0，通配ip的话，把address中的`sin_addr.s_addr`设置0，和socket()一样，返回-1表示跪了，返回0表示ok
    * tcp有一个端口重用时限机制，如果一个应用试图连续在短时间内第二次绑定同一个本地端口，会出现一个EADDRINUSE错误，等等就好了(这也行。。。)
    * 注意第二个参数，本质上是一个`struct sockaddr_in*`，被转换成了一个更加通用的`struct sockaddr*`，因为网络地址是被Socket Library所使用（这儿不太懂）
    * 不是必须在connect之前使用bind，如果一个socket没有绑定的话，库会选择本地的port和ip
* ###int socket_connect(int sockFD, struct sockaddr *servAddr, int addrLength)
    * 通过一个给定的port连接远程主机，如果这是一个流式(tcp)socket,会执行一个tcp协商(negotation)来打开一个连接，如果是数据报式(udp)socket，会把地址存储起来，之后会和某些socket操作一起作用，这里比较重要的一点就是，使用udp的情况下，不需要握手，只有根据通信时有没有数据包来判断是否连接成功，所以在通过这个socket进行读写的情况下才会报错。返回-1给跪，0ok
* ###int socket_select(int maxFDsExamined, fd_set *readFDs, fd_set *writeFDs, fd_set *exceptFDs, struct timeval *timeOut)
    * 可以轮询一个或多个socket，常用于实现非阻塞socket编程,同样error返回-1（这个函数篇幅有限，详情百度）
    * maxFDsExamined: 描述轮询的句柄范围，要比最大的socket句柄都要大,起兼容作用，windows上无所谓，可以忽略
    * readfds：（可选）指针，指向一组等待可读性检查的套接口。
    * writefds：（可选）指针，指向一组等待可写性检查的套接口。
    * exceptfds：（可选）指针，指向一组等待错误检查的套接口。
    * timeout：select()最多等待时间，对阻塞操作则为NULL，{0,0}表示立即返回，非阻塞。
* ###socket_fcntl(int sockFD, int command, int flags)
    * 可以改变一个已经打开的socketFD句柄属性，目前唯一支持的flag是`O_NONBLOCK`，可以把socket变为非阻塞模式
    * command: 等于`F_GETFL`的时候，返回这个sockFD当前的flag，等于`F_SETFL`的时候，将后面传入的flags参数设置为sockFD的flag，error时返回-1比较正统的方法如下
    

    lags = socket_fcntl(sockfd, F_GETFL,0）;
    flags |= O_NONBLOCK;
    err = socket_fcntl(sockefd, F_SETFL, flags);


* ###int socket_getpeername(int sockFD, struct sockaddr *peerAddr, int *addrLength)
    * 获取远程主机的地址，成功返回0，如果没有之前没有连接成功，GetMITLibError()会返回ENOTCONN错误，且返回-1
* ###int socket_getsockname(int sockFD, struct sockaddr *localAddr, int *addrLength)
    * 获取该socket的本地port和address，0成功，-1失败
* ###int socket_read(int sockFD, void *buffer, UInt32 numBytes)
    * 从socket中将数据读取到buffer中，numBytes应该是buffer的大小，read函数可能不会装满这个buffer，返回值是所读取数据的大小，失败返回-1，如果返回0，说明socket收到了一个EOF（比如远程主机关闭了）
    * 如果在socket上想要完整的执行读取，在读满自己希望的大小之前继续调用read，值得注意的是，如果没有数据可读的时候，read函数会阻塞，这个状态可以通过select查看到。
* ###int socket_write(int sockFD, void *buffer, UInt32 numBytes)
    * 把numBytes大小的数据从buffer中写入到socket里，同样，write可能也不会把buffer全部写入，返回值是实际写入的大小，失败返回-1
* ###int socket_readv(int sockFD, struct iovec *iov, UInt32 iovCount)
    * 等价于read，但是会把数据存放在iov数组的成员里：iov[0],iov[1]...iov[iovCount-1],iovCount的取值范围是(0,IOV_MAX]，把数据分散存放
    * struct iovec的两个成员：`caddr_t iov_base`和`int iov_len`，前者是数据存放的首地址，后者是该数据片的长度，readv函数会按顺序填充这些数据片
    * 返回值是读取数据的总大小，失败时返回-1,0是EOF
* ###int socket_recv(int sockFD, void *buffer, UInt32 numBytes, int flags)
    * 这个函数和read很像，除了最后一个参数，如果flags被设置为`MSG_DONTWAIT`，那么当数据不可用的时候这个函数会变成非阻塞。
    * 返回值是读取的数据的总数，返回-1时error
* ###int socket_send(int sockFD, void *buffer, UInt32 numBytes, int flags)
    * 和write相似，同样如果flags被设置为`MSG_DONTWAIT`时，此函数会变为非阻塞。
    * 返回值是写入的数据的总数，返回-1时error
* ###int socket_recvfrom(int sockFD, void *buffer, UInt32 numBytes, int flags, struct sockaddr *fromAddr, socklen_t *addrLength)
    * 从fromAddr指定的远程主机读取信息，要求socket必须为数据报式(udp)，numBytes是buffer的大小，不保证填满buffer，返回值是读取数据的总大小，失败时返回-1
    * 如果flags被设置为`MSG_DONTWAIT`时，此函数会变为非阻塞
    * fromAddr本质上是一个struct sockaddr_in*类型
* ###int socket_sendto(int sockFD, void *buffer, UInt32 numBytes, int flags, struct sockaddr *toAddr, socklen_t addrLength)
    * 向toAddr指定的远程主机发送信息，要求socket必须为数据报式(udp)，numBytes是要发送数据的大小，不保证全部输出buffer，返回值是读取数据的总大小，失败时返回-1
    * 如果flags被设置为`MSG_DONTWAIT`时，此函数会变为非阻塞
    * fromAddr本质上是一个struct sockaddr_in*类型
* ###int socket_shutdown(int sockFD, int howTo)
    * 单向或者双向的关闭一个连接着的socket，返回0是ok，-1是error
    * howTo：
        * `SHUT_RD`关闭正在read的端，这时socket便不再可读
        * `SHUT_WR`关闭正在write的端，这时socket会向远程主机发送一个断开的指令，告诉对方这里已经不会再write任何东西，不再可写
        * `SHUT_RDWR`关闭连接的两端，之后依旧需要使用close来释放掉socket
* ###int socket_close(int sockFD)
    * 释放掉一个socket的资源，与远程主机断开连接，这是必须要有的，不管tcp还是udp都需要执行这一步，返回0是ok，-1是error

`转载请注明出处`