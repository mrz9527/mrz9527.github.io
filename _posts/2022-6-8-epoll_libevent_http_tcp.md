# epoll

## epoll工作原理

```
1. 创建epoll，返回epoll文件描述符，可以看成是红黑树的标识符。
		底层：创建一颗红黑树。
	epoll_create(...)
	
2. 将要监听的文件描述符存放到红黑树中。
	  不想监听的文件描述符从红黑树中删除。
	  从红黑树中修改监听的文件描述符。
	epoll_ctl(...)

3. 监听事件，监听到事件后，用一个struct epoll_event数组来接收事件。
	epoll_wait(...)
```

## epoll APIs

### epoll_create

```c++
/* epoll_create。作用：创建epoll。
 参数: 
 		size: 监听的文件描述符数量，linux2.6之后，不设上限，只要大于0即可，一般设为1。
 返回值:
 		-1，失败，成功，返回创建的epoll文件描述符。
*/
int epoll_create(int size);
```

### epoll_ctl

```c++
/* epoll_ctl。添加、删除、修改监听的文件描述符。
 参数:
 		epfd: epoll的文件描述符，epoll_create(...)的返回值
 		op: 
 			 EPOLL_CTL_ADD: 添加监听的文件描述符到红黑树中。
       EPOLL_CTL_MOD: 修改监听的文件描述符到红黑树中。
       EPOLL_CTL_DEL: 从红黑树中删除监听的文件描述符。
 		fd:
 			 op操作的文件描述符。在fd上进行op操作。用于在红黑树中插入、删除、修改与fd相关的节点。
 		event: 
 			 设置在fd上的监听事件event,event.events设置读事件、写事件等等，events.data可以设置文件描述符、自定义结构体等。
 返回值:
 		0,成功; -1,失败。
*/
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

 typedef union epoll_data {
   void        *ptr; // 常用于回调，也就是reactor，需要自定义相关结构体
   int          fd;	// 不用于回调
   uint32_t     u32; // 基本不用
   uint64_t     u64; // 基本不用
 } epoll_data_t;

struct epoll_event {
  uint32_t     events;      // 要监听的事件，EPOLLIN\EPOLLOUT\EPOLLLEFT
  epoll_data_t data;        /* User data variable */
};
```

```
 说明: epoll_ctl中设置的event,当有时间发生时，会在epoll_wait中返回相应的event，但是不会单独返回fd。所以还需要在event.data中再			 次设置fd。有两种方式，方式1，直接设置event.data.fd = fd。方式2，event.data.ptr.fd = fd，ptr是自定义结构体的指针。
 			方式2更好，自定义结构体，可以填充很多东西，比如fd，还可以填充回调函数，reactor反应堆就是这种原理。
```

### epoll_wait

```c++
/*
 参数: 
 		epfd: epoll描述符，epoll_create()返回值
 		events:	用于接收监听到的事件数组
 		maxevents: 可监听的最大数组，比如设置1024，如果有超过1024个事件发生，超过的部分留到下一次epoll_wait中来接收。
 		timeout: 设置超时时间。-1表示永不超时（没有事件发生，会一直阻塞），大于0，表示超时的时间。
 返回值: 实际监听到的事件数组。
*/
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

## 为什么使用边缘触发

```
事件触发机制：
	水平触发：只要读缓冲区还有数据，就触发读事件；只要写缓冲区还可以写，就触发写事件。
	边缘触发：当读缓冲区有新的数据到来（对端发送数据时），才触发读事件；当缓冲区由满到不满时，就触发写事件。
  
边缘触发注意事项：
	读事件
			一次读完所有数据，需要while(1) + read。
			读完所有数据后，再次调用read会阻塞，所有文件描述符需要设置非阻塞。
			while退出条件,int n = read(...):
					读完全部数据退出。n == -1 && (errno == EAGIN || errno == EWOULDBLOCK)，此时break。
					客户端断开时退出。n==0时客户端断开。此时break。
          read异常时退出。n == -1 && errno != EAGIN && errno != EWOULDBLOCK，此时break
      客户端断开时退出或read异常时退出，需要关闭文件描述符，从红黑树中删除对应节点。epoll_ctl(EPOLL_CTL_DEL,...);		
```



# libevent

红黑树上树节点类型：event、bufferevent、httpevent

## event

### event工作原理

#### 主函数main

```
//-------------主流程----------------------
0. 创建并初始化监听套接字lfd
1. 创建event_base根节点
		struct event_base* base = event_base_new();
2. 初始化lfd上树节点（填充并返回ev，包括lfd、events、callback、arg)
		events = EV_READ | EV_PERSIST;
		struct event * ev = event_new(lfd, events, lfdcallback, arg);
3. 上树
		event_add(ev);
4. 循环监听和事件分发
		event_base_dispatch(base);
5. 退出清理工作
		关闭监听的套接字:	close(lfd);
		释放根节点base:	event_base_free(base);
```

#### 回调callback

##### lfdcallback

```
//-------------回调:lfdcallback----------------------
void lfdcallback(int lfd, int events, void* arg) {
	1. 获取客户端
			cfd = accept(lfd, &clientAddr, &clientAddrLen);
			
	2. 初始化cfd上树节点（填充并返回ev，包括cfd、events、callback、arg)
		events = EV_READ | EV_PERSIST;
		struct event * ev = event_new(cfd, events, cfdReadcallback, arg); // 上树时的节点ev。
	3. 上树
	event_add(ev);
}
```

##### cfdReadcallback

```
//-------------回调:cfdReadcallback----------------------
void cfdReadcallback(int cfd, int events, void* arg) {
	1. 读取数据
			char buf[1024] {0};
			int nread = read(cfd, buf, 1024);
			
			nread <=0 : 下树，并关闭客户端连接
					event_delete(ev); // ev哪里来？实际上这里下树的节点ev和上树时的ev必须是同一个，所有需要用全局数组将fd对应的节点ev保															//存起来，便于下树，在上树的时候保存，下树的时候通过判断cfd来得到ev，并下树。所以数组的元素需要包含
														// cfd和ev。
					close(cfd);
			
			nread >=0 : 
					直接write向对端发送数据？
					还是通过先下树，并重新上树，更改事件为EV_WRITE | EV_PERSIST，更改回调为cfdWritecallback。（因为没有发现可以修改树节						点的函数，没有发现event_mod函数)
}
```

#### 保存上树节点

自定义结构体event_info，用于保存红黑树上树节点，便于下树时删除节点。

```
struct event_info {
		int cfd;
		struct event* ev;
		void reset() {
				cfd = -1;
				ev = nullptr;
		}
}

#define EI_SIZE 1024
struct event_info ei[EI_SIZE];

void event_info_init() {
		for(int i = 0; i < EI_SIZE; ++i) {
				ei[i].reset();
		}
}

void event_info_add(int cfd, struct event* ev) {
		int i = 0;
		for(; i < EI_SIZE; ++i) {
				if(ei[i].cfd == 0) {
						break;
				}
		}
		
		ei[i].cfd = cfd;
		ei[i].ev = ev;
}

void event_info_del(int cfd) {
		int i = 0;
		for(; i < EI_SIZE; ++i) {
				if(ei[i].cfd == cfd) {
						break;
				}
		}
		
		ei[i].cfd = 0;
		ei[i].ev = nullptr;
}

struct event* event_info_get(int cfd) {
		int i = 0;
		for(; i < EI_SIZE; ++i) {
				if(ei[i].cfd == cfd) {
						return ei[i].ev;
				}
		}
		return nullptr;
}
```

* 完善主函数main

```
//-------------主流程----------------------

初始化ei数组
		event_info_init();

0. 创建并初始化监听套接字lfd
1. 创建event_base根节点
		struct event_base* base = event_base_new();
2. 初始化lfd上树节点（填充并返回ev，包括lfd、events、callback、arg)
		events = EV_READ | EV_PERSIST;
		struct event * ev = event_new(lfd, events, lfdcallback, arg);
3. 上树
		event_add(ev);
4. 循环监听和事件分发
		event_base_dispatch(base);
5. 退出清理工作
		关闭监听的套接字:	close(lfd);
		释放根节点base:	event_base_free(base);
```



*  完善回调函数

```
//-------------回调:lfdcallback----------------------
void lfdcallback(int lfd, int events, void* arg) {
	1. 获取客户端。cfd = accept(...)
	2. 初始化cfd上树节点。 ev = event_new(...)
	3. 上树。event_add(ev)
	
	4. 保存上树节点
		event_info_add(cfd , ev);
}
```

```
//-------------回调:cfdReadcallback----------------------
void cfdReadcallback(int cfd, int events, void* arg) {
	1. 读取数据 nread = read(...);
			nread <=0 : 下树，并关闭客户端连接
					ev = event_info_get(cfd);	// 找到cfd对应的树节点
					event_delete(ev); 				// 下树
					event_info_del(cfd);			// 清空ei数组中相应信息
					close(cfd);								// 关闭客户端文件文件描述符
			
			nread >=0 : 
					...
}
```



### event APIs

```
1. 创建event_base根节点。
struct event_base* event_base_new();

2. 释放event_base根节点。
void event_base_free(struct event_base* base);

3. 循环监听。两种方式，一般使用event_base_dispatch。
方式1：阻塞方式，没有事件发生时，内部会阻塞在epoll_wait。
	int event_base_dispatch(struct event_base* base);
方式2：flags: EVLOOP_NONBLOCK，非阻塞方式。EVLOOP_ONCE，只触发一次事件，如果事件没有触发，则阻塞等待。
	int event_base_loop(struct event_base* base, int flags);

4.退出循环监听。有两种方式，event_base_loopbreak立即退出监听。event_base_loopexit延迟退出监听。
int event_base_loopexit(struct event_base* base, const struct timeval *tv);
int event_base_loopbreak(struct event_base* base);

5. 初始化上树节点
struct event* event_new(int fd, int events, callback cb, void* arg);

6. 上树
event_add(struct event* ev);

7. 下树
event_del(struct event* ev);
```

* 其他APIs

```
查看libevent一共支持哪些多路io复用
const char **event_get_supported_methods(void);

查看当前libevent使用的是哪个多路io复用
const char *event_base_get_method(const struct event_base *);
```

## bufferevent

用户空间bufferevent中也存在缓冲区。

bufferevent_setcb可以设置三种回调，读回调、写回调和事件回调。

bufferevent回调触发时机。

读回调：当bufferevent将底层读缓冲区的数据读到自身缓冲区时，触发读回调

写回调：当bufferevent将自身写缓冲的数据写到底层缓冲区的时候，触发回调

事件回调：当超时、异常出错、断开连接时，触发事件回调。

bufferevent回调触发时机和event不同。event监视的是**内核**读写缓冲区是否可读或可写。bufferevent监视的是自身缓冲区读写到内核缓冲区的过程。

| step | event                                                        | bufferevent                                                  |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|      | 0. 初始化ei数组<br/>event_info_init();                       | 0.初始化ei数组<br>event_info_init();                         |
|      | 1. 创建event_base根节点<br/>struct event_base* base = event_base_new(); | 1. 创建event_base根节点<br/>struct event_base* base = event_base_new(); |
|      | 2. 初始化lfd上树节点event*<br>events = EV_READ  \|EV_PERSIST;<br>struct event * ev = event_new(lfd, events, lfdcallback, arg); | 2. 创建lfd上树节点bufferevent*（没有设置回调）<br>struct bufferevent * bufev = bufferevent_socket_new(base, lfd, options); |
|      | 3. 上树<br>event_add(ev);                                    | 3. (设置回调) ，并上树<br>void bufferevent_setcb(bufev,  readcb,  writecb, eventcb, cbarg);<br>上树后会自动监听事件，但是监听到事件后是否触发回调，看是否设置回调使能。 |
|      |                                                              | 4. 设置回调使能<br>使能和非使能<br>int bufferevent_enable(struct bufferevent *bufev, short event);<br>int bufferevent_disable(struct bufferevent *bufev, short event); |
|      | 4. 循环监听<br>event_base_dispatch(base);                    | 5. 循环监听<br/>event_base_dispatch(base);                   |



```
//-------------主流程----------------------
0. 创建并初始化监听套接字lfd
1. 创建event_base根节点
		struct event_base* base = event_base_new();
2. 初始化lfd上树节点（填充并返回ev，包括lfd、events、callback、arg)
		events = EV_READ | EV_PERSIST;
		struct event * ev = event_new(lfd, events, lfdcallback, arg);
3. 上树
		event_add(ev);
4. 循环监听和事件分发
		event_base_dispatch(base);
5. 退出清理工作
		关闭监听的套接字:	close(lfd);
		释放根节点base:	event_base_free(base);
```

## evhttp

```c++
// 返回请求方法
evhttp_cmd_type req_method = evhttp_request_get_command(req);

// 返回请求uri
const char* uri = evhttp_request_get_uri(req);

// 返回请求报文头，请求头是一个双向队列。
struct evkeyvalq *headers = evhttp_request_get_input_headers(req);

// 遍历请求头的每一个节点元素
for (header = headers->tqh_first; header; header = header->next.tqe_next) {
		printf("  %s: %s\n", header->key, header->value);
}

// 获取输入缓冲区（evbuffer也是队列)
struct evbuffer* buf = evhttp_request_get_input_buffer(req);

/*
// 返回buf缓冲区中的字节数 size_t size = evbuffer_get_length(buf);

// 从buf队头移除n字节，并将这n个字节拷贝到data中。
// char data[n];
// evbuffer_remove(buf, data, n);
*/

// 循环读取buf中数据
	while (evbuffer_get_length(buf)) {
  	char data[128] {0};
    evbuffer_remove(buf, data, 128);
  	// 处理data，可以打印输出
		std::cout << data << std::endl;
  }

// 填充数据到snd_buf
char snd_data[128] = "hello,world";
struct evbuffer* snd_buf;
evbuffer_add(snd_buf, snd_data, sizeof(data_len));
// 回响应包给客户端
evhttp_send_reply(req, 200, "OK", snd_buf);
```

```c++
// 解析uri，
struct evhttp_uri *decode = evhttp_uri_parse(uri);
path = evhttp_uri_get_path(decoded);
decoded_path = evhttp_uridecode(path, 0, NULL);
//
decoded_path = evhttp_uridecode(path, 0, NULL);
```



动态规划

```
dp数组的含义及下标的含义
递推公式
dp数组如何初始化
遍历顺序
打印dp数组，用于验证每一步的逻辑是否正确

```

# TCP连接及参数

## http

```
1. url解析。得到目的主机和URI资源路径
2. DNS域名解析。得到目的主机的ip。现在本地DNS服务器查，看是否有缓存，有就直接返回，没有，按照请求根服务器，（返回顶级服务器ip），访			问顶级ip，...，依次类推，知道找到目的主机ip。
3.协议栈。（报头开始标记） +  MAC头 + IP头 + TCP头 + data + FCS（结束标记）
		TCP头部：源端口、目的端口、序列号、确认号、状态位（6种，URG、ACK、PSH、RST、SYN、FIN）、窗口大小、校验和、紧急指针、选项。
```

## TCP分割数据

```
MSS:Maximum Segment Siz。最大传输单元。一个tcp包能携带的最大应用层数据data的大小。MSS = MTU - IP头 - TCP头= 1500 - 20 - 			20 = 1460Byte
MTU：Maximum Transfer Unit 最大传输单元。一个网络包的最大长度（指的ip层），以太网中一般为1500字节。MTU = IP头 + TCP头 + data 			= 1500

MSL： Maximum Segment Lifetime 报文最大生存时间 报文在网络上存在的最长时间，TCP四次挥手是主动断开连接的一方再发送完最后一个ACK			后进入TIME_WAIT状态时，需要等待2MSL时间后才变成CLOSED状态 RFC 793建议为2分钟

TTL： Time To Live 该字段指定IP包被路由器丢弃之前允许通过的最大网段数量。TTL是IPv4包头的一个8 bit字段。 
RTT： (round trip time) 客户端发送请求，服务端接收请求并返回数据，客户端接收数据。从客户端发送到客户端接收所经历的时间，称为一个来		回。一个来回的RTT很容易获取（客户端接收数据的时间戳t2 - 客户端发送数据的时间戳t1），但是网络是不稳定的，RTT的时间是变化的，需要		采集多次RTT的数据来求平均值，以及多个RTT的方差，来计算出一个合适的动态的RTT值。
RTO： Retransmission Timeout 超时重传时间。TCP中触发超时重传机制的时间，应略大于RTT。只有经历一个来回的时间，才能知道是否超时（比如syn包或者psh包丢失，或者ack包丢失），只会出现超时重传。
```

如果应用层数据大于MSS大小，就需要把数据按照MSS规格拆解成一块块，然后发送。

## 超时重传

```
// 三次握手的包丢失，重传
tcp_syn_retries: 第一个syn包丢失，发送方的重传次数
tcp_synack_retries: 第二个syn+ack包丢失，接收方的重传次数。如果syn+ack包丢失，接收方会重传syn+ack包，发送方会重传syn包，发送方每次重传syn包，都会重置										接收方的重传次数。
// 正常通信的数据包丢失，重传
tcp_retries1:先重传tcp_retries1次。
tcp_retries2:如果tcp_retries1次重传没有成功，ip层会刷新路由，并继续尝试重发，累计重发tcp_retries2次后，断开连接

// 四次挥手的包丢失，重传
tcp_orphan_retries：发送的fin包丢失，重传次数。
```

## 快速建立连接（主要针对http）

​	第一次http请求数据。先建立三次握手，在3次握手的第2个阶段，服务器发送syn+ack + cookie信息，客户端会保存cookie，并发送（ack + http GET)给服务器，就可以**接收**到**服务器**发送过来的ack+网页数据。第一次http请求，从连接建立到返回网页数据，需要等待2个RTT时间。（非快速建立连接的方式，在开启管道网络传输下，需要2.5个RTT，不开启管道网络传输，需要至少等待3个RTT，之后关闭http连接）。

​	第二次http请求数据。只要cookie未失效，第一次握手时，客户端发送syn时，附带请求数据（syn + http GET + cookie)，服务端回应ack + 网页数据。第二次http请求，从连接建立到返回网页数据，只需要等待1个RTT时间。

通过保存cookie的方式，只要cookie没有失效，就免去了三次握手。

```
tcp_fastopen:
	0,关闭快速建立连接功能
	1，作为客户端时开启该功能
	2，作为服务端时开启该功能
	3，作为客户端和服务端都开启该功能
```

## 快速重传（选择性确认SACK）

​	客户端发送多个数据包，服务端接收到的数据包是无序的，比如先接收数据包1，再接收3（请求2 + ack），再接收4（请求2+ack），再接收5（请求2+ack）。	由于服务端没有接收到数据包2，从首个乱序包（这里是3）开始，对每个接收到的乱序包都会回应（ack + 请求数据包2）。客户端发现有3次请求同一个数据包2时，会触发快速重传机制，直接传数据包2，服务端接收数据包2之后，开始请求数据包7。	

​	SACK需要客户端和服务端同时支持。

```
tcp_sack: 
	0:不开启；1：开启。默认开启。
```

## 超时重传和快速重传的区别

超时重传，发送方没有收到接收方的回应（ack），就触发超时重传机制。
快速重传，发送方收到接收方对同一个数据包的3次请求，就会触发快速重传。
超时重传时，一次比一次重传的时间长，快速重传可以弥补超时重传的缺点。

## DSACK重传

```
和sack一样，选择性重传，dsack知道为什么重传，是psh包丢了？还是ack包丢了？还是都没丢，只是网络延迟了？通过dsack就可以知道。
net.ipv4.tcp_dsack=1，linux2.4之后默认开启该功能。
```



## 窗口大小（流量控制）

发送方发送数据时，会附带自己的接收窗口大小，表示自己能接收多大的数据。这里的接收窗口大小不是真实窗口大小。

真实接收窗口大小 = 接收窗口大小 * 接收窗口比例因子，接收窗口比例因子是在三次握手过程中协商好的。如果tcpdump抓包时，没有抓到三次握手的包，那么没法计算真实的接收窗口大小。

窗口大小和tcp的缓冲区有关。

tcp接收缓冲区空间还剩余M和字节，那么真实接收窗口大小上限可以设置为M。

## 半连接队列、全连接队列

```
server收到syn，回复syn+ack时，server的tcp连接状态变为syn_recv。加入半连接队列，也叫syn队列。
client收到syn+ack,回复ack时，server的tcp连接状态变为established。从半连接队列取出，放入全连接队列，也叫accept队列。

accept()系统调用就是从accept队列中提取连接的。所以三次握手在accept()之前已经完成，accept()只是去accept队列中提取一个连接。
	
全连接队列(accept队列）最大长度？ = min(net.core.somaxconn, backlog);
net.core.somaxconn:系统层定义的最大全连接队列长度。
backlog: 应用层定义的最大全连接队列长度，backlog是listen(lfd, backlog)中的参数。

半连接队列（syn队列）最大长度？ = min(net.core.somaxconn, backlog, net.ipv4.tcp_syn_backlog);
net.ipv4.tcp_syn_backlog: 系统层定义的最大半连接队列长度。
且最大半连接队列长度不能超过最大全连接队列长度。
```

**tcp_syncooking**

```
tcp_syncooking与半连接队列相关，开启tcp_syncooking时，net.ipv4.tcp_syn_backlog不起作用.
它的作用是防止syn泛洪攻击（syn攻击）。

syn_syncooking是如何解决防洪的?
	非正常连接（泛洪）：客户端发送syn包之后关闭，服务端加入到syn队列后，并发送syn + ack，但是无响应，不会从syn队列移到accept队列，超时重传多次。当大量客户端发送syn后就关闭，导致syn队列满，accept队列空。
	当服务端syn队列满后，取出一个半连接，发送syn + ack + cookie，如果没有收到ack + cookie的回应，直接丢弃半连接，大概率是syn攻击，因为客户端发送syn后就关闭了。
	如果是正常连接，客户端收到syn + ack + cookie后，都会回一个ack + cookie，此时将半连接添加到全连接队列中。
```

## 解决syn泛洪攻击

```
优先使用方案1.
方案1. 
		对应服务端，修改重传次数tcp_synack_retries，适当减少； 修改半连接队列长度参数tcp_max_syn_backlog，适当调大； 修改半连接队列满时的参数tcp_abort_on_overflow，直接设置为1，半连接满了就直接拒绝连接，回复客户端RST包；修改
```

## tcp保活机制

```
正常建立连接后，如果客户端长时间不发送数据，会触发服务端的保活机制。
net.ipv4.tcp_keepalive_time = 7200; 保活时间是7200s，2小时内没有收到任何连接相关的活动，会启动保活机制。
net.ipv4.tcp_keepalive_intvl = 75; 每间隔75s，发送一个探测报文。
net.ipv4.tcp_keepalive_probes = 9; 探测报文发9次。
所以一个客户端长时间不发送数据，这个连接要存活2小时11分15秒。
这个时间太长，如果存在大量的这种情况，会占用服务器过多的自已，比如文件描述符、内存等。
可以通过调参来优化。
```

## time_wait及端口重用

```
如果服务端主动关闭连接（比如服务端程序退出），由于服务端time_wait需要等待2MSL时间之后，才能真正关闭连接，在time_wait阶段，服务端的监听端口是不能重用的，需要等待1分钟左右才可以重用端口。

为了使得端口可以重用，在应用程序中可以使用SOL_REUSEADDR和SOL_LINGER两种方式，在内核中可以使用tcp_tw_reuse的方式。一般在应用程序中设置，而不是更改内核的参数。理由如下：

SOL_REUSEADDR：就是服务端主动关闭时，在time_wait阶段，还是等2MSL时长，但是端口可以先拿去用，服务端重启时能直接使用该端口。
SOL_LINGER：就是服务端主动关闭时，直接发送RST包，跳过握手的四个阶段，也就是跳过了time_wait阶段。

tcp_tw_reuse：服务端主动关闭时，在time_wait阶段，端口可以先拿去用，服务端重启时能直接使用该端口，但是为了保证重启后的服务端能分辨旧的tcp连接中客户端发过来的数据包，需要再开启时间戳tcp_timestamps。

更改内核的方式有两个缺点。首先，tcp_tw_reuse是系统级的参数，对所有的tcp连接都起作用，范围太广，实际中只需要对单个应用程序加以优化。其次，由于开启了时间戳，如果客户端和服务端的系统时间不一致，可能导致客户端无法与服务端建立正常连接，会被当做过期的数据包被扔弃掉。
```

## 正常连接后，psh丢失和ack丢失的措施

```
客户端发送的psh丢失：
		超时重传、快速重传
服务端发送的ack（指的是psh的回应）丢失：
		会累计确认，在下一次接收到数据的时候回应累计ack。
```

## tcp滑动窗口

```
指的是接收窗口和发送窗口。

接收窗口大小。
A端发送数据包时，会附带A端的接收窗口大小，B端收到数据包后，会向A端回数据包，B端会根据A端的接收窗口大小来控制发送的数据大小。

发送窗口大小。
A端发送数据包给B端，B端给A端回数据包，B端的发送窗口不大于A端的接收窗口大小，不然A端会处理不过来。
```

## 路由表

pc上有路由表，路由器上也有路由表，路由表是为了确认下一跳。

```shell
# PC(10.10.10.221)的路由表
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.10.10.2      0.0.0.0         UG    0      0        0 ens33
10.10.10.0      0.0.0.0         255.255.255.0   U     0      0        0 ens33

# Destination：目的ip所在的网络号
# Gateway：网关
# Genmask：子网掩码
# Iface： 网卡
# Flags: U表示该路由有效（up），G表示需要通过网关来转发数据包

# 特殊分析
# 0.0.0.0目的网络：表示目的ip在路由表中匹配不到网络时，就走默认0.0.0.0网络所在的网关gateway
# 0.0.0.0网关：表示没有网关，一般是目的ip和源ip在同一个网络中，就不需要走网关，只需要走交换机交换数据就可，所有网关为0或空
```

```
分析1：pcA（src_ip：10.10.10.221）向pcA(dst_ip：10.10.10.100）发送数据包
pcA查看自己的路由表。dst_ip & genmask，看看结果是否在Destination中出现，出现表示匹配到路由，就走相应的网关Gateway转发
10.10.10.100 & 0.0.0.0 = 0.0.0.0
10.10.10.200 & 255.255.255.0 = 10.10.10.0

两条路由都能匹配，优先匹配Destination为10.10.10.0的目的网络，对应的网关为0.0.0.0，表示不需要网关。
src_ip和dst_ip确实在同一个网络10.10.10.0中，只需要交换机就可以了。
```

```
分析2：pcA（src_ip：10.10.10.221）向pcA(dst_ip：110.242.68.4）发送数据包
pcA查看自己的路由表。dst_ip & genmask，看看结果是否在Destination中出现，出现表示匹配到路由，就走相应的网关Gateway转发
110.242.68.4 & 0.0.0.0 = 0.0.0.0
110.242.68.4 & 255.255.255.0 = 110.242.68.0

只有第一条目能匹配，优先匹配Destination为0.0.0.0的目的网络，对应的网关为10.10.10.2，表示通过网关10.10.10.2走出大门，和外面的网络进行交互。至于10.10.10.2的路由表是什么，10.10.10.2的下一跳应该选什么，需要查看10.10.10.2的路由表，这里不在给出。
```

## 路由问题解决

```
先画出路由拓扑图。再分析路由表，即可解决ping不通的问题。
ping不通的情况，需要添加静态路由。
```

## http压力测试

wrk

## syn泛洪测试

hing3



# zeromq

只有bind、recv、send、connect，没有accept和listen，zmq底层已经把accept和listen封装进去了。

```
请求req、发布pub、推push，对应的是send()，发送数据。
回应rep、订阅sub、拉pull，对应的是recv()，接收数据。
req / rep
pub / sub
push / pull
```

## req/rep模式

伪代码

```c++
//服务端回应rep
		int main() {
				// 1.创建sock。 
      	zmq::context_t ctxt (1);
      	zmq::socket_t reper(ctxt, ZMQ_REP);
      	// 2.绑定端口。
      	sock.bind("tcp://*:5555");
      	// 3. 循环
        while(1) {
          	// 4.接收客户端请求
            zmq::message_t rep_msg;
            sock.recv(&rep_msg);
            // 4. 处理rep_msg
          	...
            
            // 5. 回应数据
            zmq::message_t rep_msg(100);
          	// 填充数据
          	memcpy(rep_msg.data(), "xxxxx", 100);
            sock.send(req_msg);
          	
          	// 疑问：如果有多个客户端请求过来，服务端怎么判断这个请求是哪个客户端的，也就是处理后的数据要发给哪个客户端？
          	// 搞清楚之后，可以开个线程池，将上述处理过程放入线程池中。
        }
		}

//客户端请求req
		int main() {
      	// 1.创建sock。 
      	zmq::socket_t reper(ZMQ_REQ);
      	// 2.绑定端口。
      	sock.bind("tcp://*:5555");
      	// 3. 循环
        while(1) {
          	// 4.接收客户端请求
            zmq::message_t rep_msg;
            sock.recv(&rep_msg);
            // 4. 处理rep_msg
          	...
            
            // 5. 回应数据
            zmq::message_t rep_msg;
            sock.send(req_msg);
          	
          	// 疑问：如果有多个客户端请求过来，服务端怎么判断这个请求是哪个客户端的，也就是处理后的数据要发给哪个客户端？
          	// 搞清楚之后，可以开个线程池，将上述处理过程放入线程池中。
        }
    }
```

rpc 、 十大排序算法再过一遍，动态规划