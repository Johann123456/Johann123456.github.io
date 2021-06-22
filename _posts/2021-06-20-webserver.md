---
layout:     post
title:      "webserver"
subtitle:   "tinywebserver"
date:       "2021-06-20"
author:     "Johann"
# # header-img: "img/post-bg-js-version.jpg"
tags:
    - 项目
---
# 轻量级webserver
## IO多路复用
### select 
在一段时间内，监听用户感兴趣的文件描述符上的可读可写和异常等事件
```C++
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```
- nfds是监听的文件描述符总数，设定为监听的文件描述符中的最大值+1
- readfds、writefds、exceptfds是文件描述符集合，传入参数
- timeout是超时时间，如果设定为0则select立即返回，设置为NULL永远阻塞，设置为timeval则是等待一定时间
- 返回值，成功时返回就绪描述符总数，超时时间内没有就绪的返回0，失败返回-1
### poll
与select类似，在一定时间内轮询一定数量的文件描述符，查看是否有就绪的
```C++
int poll(struct pollfd* fds, nfds_t, int timeout);
//fds即是输入又是输入
struct pollfd
{
    int fd;//文件描述符
    short events;//注册事件
    short revents;//内核写入，实际发生的事件
}
```
### epoll
epoll是一组函数完成任务
epoll针对select的问题进行了改进：
- select传入fd数组，需要拷贝一份到内核，高并发场景下会消耗资源
- select在内核中仍是遍历的方式检查文件描述符的就绪状态
- select仅仅返回就绪的文件描述符的个数，具体哪个可读需要用户自己遍历

epoll的改进：
- 内核保存一份文件描述符集合，无需用户每次都传入，只需告诉内核修改的部分即可
- 内核不用轮询的方式找到就绪的文件描述符，而是通过异步IO事件唤醒
- 内核仅将有IO事件的文件描述符返回给用户，用户无需遍历整个文件描述符集合

```C++
//epoll的三个函数
int epoll_create(int size);//创建一个内核事件表的文件描述符
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);//操作内核事件表epfd，op可以有添加修改注册，
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);//返回就绪的文件描述符的个数
//epoll_event* events是输出参数，用于输出就绪的事件
struct epoll_event{
    __unit32_t events;//epoll事件
    epoll_data_t data;//用户数据
}
//所以输出的events可以直接用events[i].events判断事件，用events[i].data.fd来返回就绪的描述符
```
### epoll的LT和ET模式
LT是电平触发，当epoll_wait()检测到有事件并通知应用程序后，应用程序可以不立刻处理事件，应用程序下一次调用epoll_wait()时，epoll_wait()还会告知应用程序此事件，直到该事件被处理。
ET是边沿触发，当epoll_wait()检测到事件，应用程序必须立刻处理该事件，后续的epoll_wait()将不会向应用程序通知这一事件。
### 三种IO复用的比较
- select和poll每次都返回整个用户注册的事件集合，所以应用程序索引就绪文件描述符的时间复杂度是O(n)，epoll返回的是就绪的事件，应用程序索引就绪文件描述符的时间复杂度为O(1)。
- select和poll只能工作在低效的LT模式，epoll可以工作在ET模式，并且支持EPOLLONESHOT事件（注册了该事件的描述符，操作系统只能出发一个事件，确保一个描述符只由一个线程来处理）
- 但是如果活动连接比较多，epoll效率未必高，因为内核检测到就绪的文件描述符就调用**回调函数**把该文件描述符上的事件写到就绪事件队列中，内核在合适的时间把这个队列从内核拷贝到用户空间，如果活动连接较多，回调函数触发的太过频繁，所以epoll适用于连接数量多，但是活动连接较少的情况。
## 线程池
线程池的目的是为了减少线程创建和消除的开销
## 