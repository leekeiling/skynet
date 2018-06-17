### skynet基本结构 ---- skynet_context管理模块

我们创建一个新的服务，首先要先找到对应服务的module，在创建完module实例并完成初始化以后，还需要创建一个skynet_context上下文，并将module实例和module模块和这个context关联起来，最后放置于skynet_context list中，一个个独立的沙盒环境就这样被创建出来了，下面来看主要的数据结构：

**创建新服务步骤：**

**1、对应服务的module实例化和初始化**

**2、创建skynet_context上下文**

**3、将module实例和module模块与skynet_context关联**

**4、skynet-context list**

```c++
// skynet_server.c
struct skynet_context {
    void * instance;                // 由指定module的create函数，创建的数据实例指针，同一类服务可能有多个实例，
                                    // 因此每个服务都应该有自己的数据
        
    struct skynet_module * mod;     // 引用服务module的指针，方便后面对create、init、signal和release函数进行调用
    void * cb_ud;                   // 调用callback函数时，回传给callback的userdata，一般是instance指针
    skynet_cb cb;                   // 服务的消息回调函数，一般在skynet_module的init函数里指定
    struct message_queue *queue;    // 服务专属的次级消息队列指针
    FILE * logfile;                 // 日志句柄
    char result[32];                // 操作skynet_context的返回值，会写到这里
    uint32_t handle;                // 标识唯一context的服务id
    int session_id;                 // 在发出请求后，收到对方的返回消息时，通过session_id来匹配一个返回，对应哪个请求
    int ref;                        // 引用计数变量，当为0时，表示内存可以被释放
    bool init;                      // 是否完成初始化
    bool endless;                   // 消息是否堵住
    
    CHECKCALLING_DECL
};
    
// skynet_handle.c
// 这个结构用于记录，服务对应的别名，当应用层为某个服务命名时，会写到这里来
struct handle_name {
    char * name;                   // 服务别名
    uint32_t handle;               // 服务id
};
    
struct handle_storage {
    struct rwlock lock;            // 读写锁
    
    uint32_t harbor;               // harbor id
    uint32_t handle_index;         // 创建下一个服务时，该服务的slot idx，一般会先判断该slot是否被占用，后面会详细讨论
    int slot_size;                 // slot的大小，一定是2^n，初始值是4
    struct skynet_context ** slot; // skynet_context list
        
    int name_cap;                  // 别名列表大小，大小为2^n
    int name_count;                // 别名数量
    struct handle_name *name;      // 别名列表
};
    
static struct handle_storage *H = NULL;
```

我们创建一个新的skynet_context时，会往slot列表中放，当一个**消息送达一个context时**，其**callback函数就会被调用**，**callback函数一般在module的init函数里指定，调用callback函数时，会传入userdata（一般是instance指针），source（发送方的服务id），type（消息类型），msg和sz（数据及其大小），每个服务的callback处理各自的逻辑。这里其实可以将modules视为工厂，而skynet_context则是该工厂创建出来的实例，而这些实例，则是通过handle_storage来进行管理。**