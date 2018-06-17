### Skynet 基本数据结构----modules管理模块

#### **modules管理模块**

我们所写的C服务在编译成**so库**以后，会在某个时机被加载到一个**modules的列表**中，当要创建该类**服务的实例**时，将从modules列表取出该**服务的函数句柄**，调用**create函数创建服务实例**，并且**init**之后，将**实例赋值给一个新的context对**象后，注册到图1所示的**skynet_context list**中，**一个新的服务就创建完成了**。我们存放modules的模块数据结构如下所示：

```c++
// skynet_module.h
typedef void * (*skynet_dl_create)(void);
typedef int (*skynet_dl_init)(void * inst, struct skynet_context *, const char * parm);
typedef void (*skynet_dl_release)(void * inst);
typedef void (*skynet_dl_signal)(void * inst, int signal);
    
struct skynet_module {
    const char * name;          // C服务名称，一般是C服务的文件名
    void * module;              // 访问该so库的dl句柄，该句柄通过dlopen函数获得
    skynet_dl_create create;    // 绑定so库中的xxx_create函数，通过dlsym函数实现绑定，调用该create即是调用xxx_create
    skynet_dl_init init;        // 绑定so库中的xxx_init函数，调用该init即是调用xxx_init
    skynet_dl_release release;  // 绑定so库中的xxx_release函数，调用该release即是调用xxx_release
    skynet_dl_signal signal;    // 绑定so库中的xxx_signal函数，调用该signal即是调用xxx_signal
};
    
// skynet_module.c
#define MAX_MODULE_TYPE 32
    
struct modules {
    int count;                  // modules的数量
    struct spinlock lock;       // 自旋锁，避免多个线程同时向skynet_module写入数据，保证线程安全
    const char * path;          // 由skynet配置表中的cpath指定，一般包含./cservice/?.so路径
    struct skynet_module m[MAX_MODULE_TYPE];  // 存放服务模块的数组，最多32类
};
    
static struct modules * M = NULL;
```

通过上面的注释，我们大概可以了解skynet_module结构的作用了，也就是说一个符合规范的**skynet c服务，应当包含create，init，signal和release四个接口**，在该c服务编译成so库以后，在程序中动态加载到skynet_module列表中，这里通过**dlopen函数来获取so库的访问句柄，并通过dlsym将so库中对应的函数绑定到函数指针中**，对于两个函数的说明如下所示：

```
// 引证来源：https://linux.die.net/man/3/dlopen
void *dlopen(const char *filename, int flag);
void *dlsym(void *handle, const char *symbol);
    
dlopen()
The function dlopen() loads the dynamic library file named by the null-terminated string filename and returns
an opaque "handle" for the dynamic library...
    
dlsym()
The function dlsym() takes a "handle" of a dynamic library returned by dlopen() and the null-terminated symbol  
name, returning the address where that symbol is loaded into memory...
```

**dlopen函数，本质是将so库加载内存中，并返回一个可以访问该内存块的句柄**，**dlsym，则是通过该句柄和指定一个函数名，到内存中找到指定函数**，在内存中的地址，这里将该地址赋值给skynet_module中的create、init、signal或release的其中一个函数指针（根据传入的symbol名称，即函数名来定），一个模块被加载以后，将被放置到modules的skynet_module数组中，当要创建该module的实例时，将会从skynet_module中取出对应的模块，并调用create函数创建实例，然后将实例指针传入init函数完成初始化以后，赋值给context。
一个C服务，定义以上四个接口时，一定要以文件名作为前缀，然后通过下划线和对应函数连接起来，因为skynet加载的时候，就是通过这种方式去寻找对应函数的地址的，比如一个c服务文件名为logger，那么对应的4个函数名则为logger_create、logger_init、logger_signal、logger_release。

