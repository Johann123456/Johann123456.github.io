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
## 单例模式
只创建一次对象
```C++
class singleton{
private:
    singleton();
    singleton(const singleton& other);
public:
    static singleton* getinstance();
    static singleton* m_instance;
}
singleton* singleton::m_instance = nullptr;
//线程非安全版本
singleton* singleton::getinstance(){
    if(m_instance == nullptr){
        m_instance = new singleton();
    }
    return m_instance;
}
//由于多线程情况下可能会有多个线程同时执行判断，可能会创建多个对象
//线程安全版本，加锁
singleton* singleton::getinstance(){
    locker lock;
    if(m_instance == nullptr){
        m_instance = new singleton();
    }
    return m_instance;
}
//加锁之后只有一个线程可以获得锁，保证了多线程下的安全，但是代价过高，因为当一个线程获取了锁创建了对象之后，其他线程只是在进行读操作即只是执行return m_instance所以加锁是没有必要的，所以锁会造成资源浪费
//线程安全版本，双检查锁
singleton* singleton::getinstance(){
    if(m_instance == nullptr){
        locker lock;
        if(m_instance == nullptr){
            m_instance = new singleton();
        }        
    }
    return m_instance;
}
//如果是空，有线程进来获得锁，然后创建对象，锁后检查原因：其他线程如果有两个同时进来，如果没有检查if，那么会先后获得锁然后创建对象，如果有了后面的if就可以避免这种情况
//但是这种方法是不安全的，内存读写会有reorder，会导致双检查锁失效，`m_instance = new singleton()`正常情况下指令序列是：分配内存、调用构造函数（对分配的内存进行初始化）、返回值（把地址指针给m_instance），但是reoder之后可能是先分配内存然后返回值然后调用构造函数，这就会导致一个线程在调用构造器之前m_instance就不是null，而另一个线程进来之后会判断认为这是一个合法的对象而直接使用
//解决方法C++11使用atomic_thread_fence()来禁止reorder
std::aotmic<singleton*> singleton::m_instance;
std::mutex singleton::m_mutex;

singleton* singleton::getinstance(){
    singleton* tmp = m_instance.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);
    if(tmp == nullptr){
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if(tmp == nullptr){
            tmp = new singleton();
            std::atomic_thread_fence(std::memory_order_release);
            m_instance.store(tmp, std::memory_order_relaxed);
        }        
    }
    return tmp;
}

```

## 有限状态机
http头部未提供长度字段，并且头部变化很大，根据http协议规定，判断头部结束的依据是遇到**空行**，空行仅包含回车换行符。因此每完成一次读操作就要分析新读入的数据中是否有空行，同时可以完成对请求头部的解析（请求行和头部域）
```C++
//请求行
GET http://ww.baidu.com/index.html HTTP/1.0 
//头部字段
User-Agent: Wget/1.12 (linux-gnu)
Host: www.baidu.com
Connection: close
```
解析http请求使用的是主从状态机，主状态机有两种状态CHECK_STATE_REQUESTLINE表示当前分析的是请求行，CHECK_STATE_HEADER是分析头部字段，从状态机根据从buffer读取的状态可以分为LINE_OK读取完成一行（遇到“/r/n”），没有读完一行LINE_OPEN，读取到单个的换行符LINE_BAD三种状态，主状态机调用从状态机来读一行，并进行分析，分析完请求行会把状态改成check_state_header实现状态转移。
## 高并发模式
半同步/半异步模式