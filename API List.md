API List

以下列出了所有 skynet 目前提供的 lua 模块的 API 列表，细节待补充：

**skynet** [LuaAPI](https://github.com/cloudwu/skynet/wiki/LuaAPI)

**构建服务需要的 API ：**

- **register_protocol(class)** 在当前服务类注册一类消息的处理机制。
- **start(func)** 用 func 函数初始化服务，**并将消息处理函数注册到 C 层**，让该服务可以工作。
- **init(func)** 若服务尚未初始化完成，则注册一个函数等服务初始化阶段再执行；若服务已经初始化完成，则立刻运行该函数。
- **dispatch(type, func)** 为 type 类型的消息设定一个处理函数。
- **getenv(key)** 返回当前进程内注册表项的值。
- **setenv(key,value)** 向当前进程内注册表添加一项(不可以重置已有配置项）。
- **memlimit(bytes)** 设定当前服务最多可以使用多少字节的内存，该函数必须先于 start 函数调用。

**构建框架需要的 API ：**

- **newservice(name, ...)** 启动一个名为 name 的新服务。
- **uniqueservice(name, ...)** 启动一个唯一服务，如果服务该服务已经启动，则返回已启动的服务地址。
- **queryservice(name)** 查询一个由 uniqueservice 启动的唯一服务的地址，若该服务尚未启动则等待。
- **localname(name)** **返回同一进程内，用 register 注册的具名服务的地址。**

**任务调度和自身控制 API ：**

- **sleep(time)** 让当前的任务等待 time * 0.01s 。
- **yield()** 让出当前的任务执行流程，使本服务内其它任务有机会执行，随后会继续运行。
- **wait()** 让出当前的任务执行流程，直到用 wakeup 唤醒它。
- **wakeup(co)** 唤醒用 wait 或 sleep 处于等待状态的任务。
- **fork(func, ...)** 启动一个新的任务去执行函数 func 。
- **timeout(time, func)** 设定一个定时触发函数 func ，在 time * 0.01s 后触发。
- **starttime()** 返回当前进程的启动 UTC 时间（秒）。
- **now()** 返回当前进程启动后经过的时间 (0.01 秒) 。
- **time()** 通过 starttime 和 now 计算出当前 UTC 时间（秒）。
- **address(addr)** 将一个服务地址转换为一个可供显示的字符串。
- **self()** 返回当前服务的地址。
- **exit()** 结束当前服务。

**状态查询 API ：**

- **info_func(func)** 注册一个查询内部状态的函数，供 debug 协议调用。
- **stat(what)** 查询当前服务的内部状态，what 可以是
  - endless 是否处于长期占用 cpu 的状态
  - mqlen 尚未处理的消息条数
  - message 已处理的消息条数
  - cpu占用总 cpu 时间

**消息传播 API ：**

- **call(addr, type, ...)** 用 type 类型发送一个消息到 addr ，并等待对方的回应。
- **send(addr, type, ...)** 用 type 类型向 addr 发送一个消息。
- **redirect(addr, source, type, ...)** 伪装成 source 地址，向 addr 发送一个消息。
- **ret(msg, sz)** 将打包好的消息回应给当前任务的请求源头。
- **retpack(...)** 将消息用 pack 打包，并调用 ret 回应。
- **response([packfunc])** 生成一个回应函数，用于在将来回应当前任务。当消息不使用默认的 lua 类型时，需提供对应的消息打包函数。
- **error(msg)** 向 log 服务发送一条消息。
- **pack(...)** 用默认的 lua 类型打包一个消息，返回可供内部调用的指针和长度。
- **packstring(...)** 用默认的 lua 类型打包一个消息，返回一个 lua 字符串。
- **unpack(msg [, sz])** 将 pack 或 packstring 打包的消息解包，若没有 sz 则 msg 必须为一个 lua 字符串 。
- **tostring(msg, sz)** 将一个指针和长度定义的消息包转换成一个字符串。

**用于某些特殊需要，只有在明确知道其意义再使用的 API ：**

- **harbor(addr)** 查询一个服务地址在哪个节点上。
- **rawsend(addr, type, msg, sz**) 用 type 类型向 addr 发送一个打包好的消息。
- **rawcall(addr, type, msg, sz)** 用 type 类型发送一个已经打包好的消息到 addr ，并等待对方的回应，但不对回应信息解包。
- **trash(msg, sz)** 销毁一个指针和长度定义的消息包。
- **dispatch_unknown_request(func)** 为无法处理的消息类型设定一个处理函数。
- **dispatch_unknown_response(func**) 为无法处理的回应消息设定一个处理函数。

**内部使用或由于兼容目的存在的 API ：**

- **genid()** 生成唯一 session 。
- **dispatch_message(typeid, msg, sz, session, source) 默认的消息处理过程，由 C 层传递给它消息的五元组：消息类型 id 、指针、长度、session 号、消息源地址。**
- **pcall(func, ...)** 执行一个函数，捕获可能抛出的异常，并保证在此之前运行由 init 注册的初始化过程。
- **init_service**(func) 用 func 函数初始化服务。
- **endless()** 查询当前服务的当前任务是否处于长期占用 cpu 的状态。
- **mqlen()** 查询当前服务有多少尚未处理的消息。
- **task(result)** 返回当前服务尚未处理完成的任务。
- **term(source**) 假定 source 服务已经退出。

**通过导入 skynet.manager 的附加 API ，由于兼容目的或构建框架的目的而存在：**

- **launch(name, ...)** 直接启动一个 C 服务。
- **kill(addr)** 强制退出一个服务。
- **abort()** 结束 skynet 进程。
- **register(name)** 给当前服务起一个字符串名。
- **name(name, address)** 为 address 指定的服务起一个名字。
- **forward_type(map, start_func)** 为当前服务注册一个以消息转发为目的的消息处理函数。
- **filter(filter_func, start_func)** 为当前服务注册一个消息处理函数，并在每条消息前注册一个消息过滤器。
- **monitor(address)** 注册一个全局服务监控服务，监控所有服务的退出事件。

**skynet.cluster** [Cluster](https://github.com/cloudwu/skynet/wiki/Cluster)

- **call(node, address, ...**) 向一个节点上的一个服务提起一个请求，等待回应。
- **send(node, address, ...)** 向一个节点上的一个服务推送一条消息。
- **open(port)** 让当前节点监听一个端口。
- **reload([config]**) 重新加载当前节点的网络配置，可以提供一个 config 表，取代文件配置。
- **proxy(node, address)** 为远程节点上的服务创建一个本地代理服务。
- **snax(node, name, address)** 向远程节点上的 snax 服务创建一个本地代理。
- **register(name, address)** 在当前节点上为一个服务起一个字符串名字，之后可以用这个名字取代地址。
- **query(node, name)** 在远程节点上查询一个名字对应的地址。

**skynet.coroutine** [Coroutine](https://github.com/cloudwu/skynet/wiki/Coroutine)

用于取代 lua 默认的 coroutine 库，以和 skynet 框架配合。新增 api 有：

- thread(co) 返回一个 coroutine 所属的 skynet 任务。

**skynet.codecache** [CodeCache](https://github.com/cloudwu/skynet/wiki/CodeCache)

**skynet.profile** [Profile](https://github.com/cloudwu/skynet/wiki/Profile)

**skynet.datacenter** [DataCenter](https://github.com/cloudwu/skynet/wiki/DataCenter)

**skynet.harbor** [Cluster](https://github.com/cloudwu/skynet/wiki/Cluster)

**skynet.multicast** [Multicast](https://github.com/cloudwu/skynet/wiki/Multicast)

**skynet.queue** [CriticalSection](https://github.com/cloudwu/skynet/wiki/CriticalSection)

**skynet.sharedata** [ShareData](https://github.com/cloudwu/skynet/wiki/ShareData)

**skynet.socket** [Socket](https://github.com/cloudwu/skynet/wiki/Socket)

**skynet.dns** [Socket](https://github.com/cloudwu/skynet/wiki/Socket)

**skynet.socketchannel** [SocketChannel](https://github.com/cloudwu/skynet/wiki/SocketChannel)

待补充：

**skynet.db.redis**

**skynet.db.mongo**

**skynet.db.mysql**

**bson**

**sproto**

**skynet.crypt**

**skynet.datasheet**

**http**