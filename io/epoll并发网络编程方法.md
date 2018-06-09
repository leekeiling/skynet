## epoll并发网络编程方法

### select方法的缺点：

1. 最大并发数限制
2. select每次调用都会线性扫描全部的FD集合。
3. 内核/用户空间 内存拷贝问题，如何让内核把FD消息通知给用户空间呢？在这个问题上select采取了内存拷贝方法。

### epoll方法的优点

1. 没有并发数限制
2. 只管活跃的连接，跟连接总数无关
3. 内存拷贝，使用“共享内存”

### epoll的api

1. int epoll_create(int size)

   生成一个epoll专用的文件描述符，其实是申请一个内核空间，监控socket fd的事件

2. int epoll_ctl(int epfd, int op, int fd, struct eopll_event * event)

   控制某个epoll文件描述符上的事件：注册、修改、删除。相对于select模型中的FD_SET和FD_CLR宏

   epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。

   第一个参数是epoll_create()的返回值；

   第二个参数表示动作，用三个宏来表示：

   **EPOLL_CTL_ADD**：注册新的fd到epfd中；

   **EPOLL_CTL_MOD**：修改已经注册的fd的监听事件；

   **EPOLL_CTL_DEL**：从epfd中删除一个fd；

   第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：

   struct epoll_event {

      __uint32_t events;      // Epollevents

      epoll_data_t data;      // Userdatavariable

   };

   events可以是以下几个宏的集合：

   **EPOLLIN** ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；

   **EPOLLOUT**：表示对应的文件描述符可以写；

   **EPOLLPRI**：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；

   **EPOLLERR**：表示对应的文件描述符发生错误；

   **EPOLLHUP**：表示对应的文件描述符被挂断；

   **EPOLLET**：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。

   **EPOLLONESHOT**：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。

   typedef union epoll_data {

      void *ptr;

   ​    int fd;

      __uint32_t u32;

      __uint64_t u64;

   } epoll_data_t;

   可见epoll_data是一个union结构体,借助于它应用程序可以保存很多类型的信息:fd、指针等等。有了它，应用程序就可以直接定位目标了。

3. int epoll_wait(int epfd, int op, struct epoll_event* events, int maxevents, int timeout)

   等待I/O事件的发生，相对于select模型中的select函数，返回发生事件数。如返回0表示已超时。参数说明：

   **epfd**:由epoll_create() 生成的epoll专用的文件描述符；

   **epoll_event**:用于回传代处理事件的数组；从内核得到事件的集合；

   **maxevents**:每次能处理的事件数；告之内核这个events有多大，这个 maxevents的值必须大于0；

   **timeout**:等待I/O事件发生的超时值；（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞；

   ​

### 关于ET、LT两种工作模式

ET模式仅当状态发生变化的时候才获得通知,这里所谓的状态的变化并不包括缓冲区中还有未处理的数据,也就是说,如果要采用ET模式,需要一直read/write直到出错为止.

LT模式是只要有数据没有处理就会一直通知下去的.

### epoll基本用法

1. epoll_create创建一个epoll句柄
2. 网络主循环里面，每一帧的调用epoll_wait来查询所有的网络接口，看哪一个可读，哪一个可写
3. close()关闭epoll句柄

### epoll编程框架

```c++
for(;;){
	nfds = epoll_wait(epfd,events,20,500);
	for(i=0;i<nfds;++i){
		if(events[i].data.fd==listenfd) //有新的连接 
		{
			connfd = accept(listenfd,(sockaddr*)&clientaddr, &client);//accept这个连接
			ev.data.fd=connfd;
			ev.events=EPOLLIN|EPOLLET;
			epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);//将新的fd添加到epoll的监听队列中 
		} 
		else if(events[i].events&EPOLLIN)//接收到数据，读socket
		{
			n = read(sockfd,line,MAXLINE) //读
			ev.data.ptr = md; //md为自定义类型，添加数据
			ev.events=EPOLLOUT|EPOLLET;
			epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识	，等待下一个循环时发送数据，一般处理的精髓 
		} 
		else if(events[i].events&EPOLLOUT)//有数据待发送，写socket
		{
			struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;//取数据 
			sockfd = md->fd;
			send(sockfd, md->ptr,strlen((char*)md->ptr),0);//发送数据
			ev.data.fd=sockfd;
			ev.events=EPOLLIN|EPOLLET;
			epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时接收数据 
		} 
		else{
			//to do 
		} 
	}
} 
```

