## Socket

**skynet 的 C API 采用异步读写，你可以使用 C 调用，监听一个端口，或发起一个 TCP 连接。但具体的操作结果要等待 skynet 的事件回调。skynet 会把结果以 `PTYPE_SOCKET` 类型的消息发送给发起请求的服务**。（参考skynet_socket.h）

在处理实际业务中，这样的 API 很难使用，所以又提供了一组阻塞模式的 lua API 用于 TCP socket 的读写。它是对 C API 的封装。

**所谓阻塞模式，实际上是利用了 lua 的 coroutine 机制。当你调用 socket api 时，服务有可能被挂起（时间片被让给其他业务处理)，待结果通过 socket 消息返回，coroutine 将延续执行。**

```
local socket = require "skynet.socket"
```

这样就可以在你的服务中引入这组 api 。

- **socket.open(address, port)** 建立一个 TCP 连接。返回一个数字 id 。
- **socket.close(id)** 关闭一个连接，这个 API 有可能阻塞住执行流。因为如果有其它 coroutine 正在阻塞读这个 id 对应的连接，会先驱使读操作结束，close 操作才返回。
- **socket.close_fd(id)** 在极其罕见的情况下，需要粗暴的直接关闭某个连接，而避免 socket.close 的阻塞等待流程，可以使用它。
- **socket.shutdown(id)** 强行关闭一个连接。和 close 不同的是，它不会等待可能存在的其它 coroutine 的读操作。一般不建议使用这个 API ，但如果你需要在 `__gc` 元方法中关闭连接的话，shutdown 是一个比 close 更好的选择（因为在 gc 过程中无法切换 coroutine）。
- **socket.read(id, sz)** 从一个 socket 上读 sz 指定的字节数。如果读到了指定长度的字符串，它把这个字符串返回。如果连接断开导致字节数不够，将返回一个 false 加上读到的字符串。如果 sz 为 nil ，则返回尽可能多的字节数，但至少读一个字节（若无新数据，会阻塞）。
- **socket.readall(id)** 从一个 socket 上读所有的数据，直到 socket 主动断开，或在其它 coroutine 用 socket.close 关闭它。
- **socket.readline(id, sep)** 从一个 socket 上读一行数据。sep 指行分割符。默认的 sep 为 "\n"。读到的字符串是不包含这个分割符的。
- **socket.block(id)** 等待一个 socket 可读。

socket api 中有两个不同的**写操作**。对应 skynet 为每个 socket 设定的两个写队列。通常我们只需要用：

- **socket.write(id, str)** 把一个字符串置入正常的写队列，skynet 框架会在 socket 可写时发送它。

但同时 skynet 还提供一个低优先级的写操作（如果你不需要这个设计，可以不使用它）：

- **socket.lwrite(id, str)** 把字符串写入低优先级队列。如果正常的写队列还有写操作未完成时，低优先级队列上的数据永远不会被发出。只有在正常写队列为空时，才会处理低优先级队列。但是，每次写的字符串都可以看成原子操作。不会只发送一半，然后转去发送正常写队列的数据。

对于服务器，通常我们需要**监听一个端口，并转发某个接入连接的处理权**。那么可以用如下 API ：

- **socket.listen(address, port)** 监听一个端口，返回一个 id ，供 start 使用。
- **socket.start(id , accept)** accept 是一个函数。每当一个监听的 id 对应的 socket 上有连接接入的时候，都会调用 accept 函数。这个函数会得到接入连接的 id 以及 ip 地址。你可以做后续操作。

**每当 accept 函数获得一个新的 socket id 后，并不会立即收到这个 socket 上的数据。这是因为，我们有时会希望把这个 socket 的操作权转让给别的服务去处理。**

**socket 的 id 对于整个 skynet 节点都是公开的。也就是说，你可以把 id 这个数字通过消息发送给其它服务，其他服务也可以去操作它。任何一个服务只有在调用 socket.start(id) 之后，才可以收到这个 socket 上的数据。skynet 框架是根据调用 start 这个 api 的位置来决定把对应 socket 上的数据转发到哪里去的。**

**向一个 socket id 写数据也需要先调用 start ，但写数据不限制在调用 start 的同一个服务中。也就是说，你可以在一个服务中调用 start ，然后在另一个服务中向其写入数据。skynet 可以保证一次 write 调用的原子性。即，如果你有多个服务同时向一个 socket id 写数据，每个写操作的串不会被分割开**。

- **socket.abandon(id)** 清除 socket id 在本服务内的数据结构，但并不关闭这个 socket 。这可以用于你把 id 发送给其它服务，以转交 socket 的控制权。
- **socket.warning(id, callback)** 当 id 对应的 socket 上待发的数据超过 1M 字节后，系统将回调 callback 以示警告。`function callback(id, size)` 回调函数接收两个参数 id 和 size ，size 的单位是 K 。如果你不设回调，那么将每增加 64K 利用 skynet.error 写一行错误信息。

### UDP

**skynet 为 udp 协议做了有限的支持。和 tcp 协议不同，udp 协议不需要阻塞读取。这是因为 udp 是不可靠协议，无法预期下一个读到的数据包是什么（协议允许乱序和丢包）。所以 skynet 的 udp 协议封装采用的是 callback 的方式。**

**你可以用 `socket.udp` 这个 API 创建一个 udp handle ，并给它绑定一个 callback 函数。当这个 handle 收到 udp 消息时，callback 函数将被触发。**

**socket.udp(function(str, from), address, port) : id**

- 第一个参数是一个 callback 函数，它会收到两个参数。str 是一个字符串即收到的包内容，from 是一个表示消息来源的字符串用于返回这条消息（见 socket.sendto）。
- 第二个参数是一个字符串表示绑定的 ip 地址。如果你不写，默认为 ipv4 的 0.0.0.0 。
- 第三个参数是一个数字， 表示绑定的端口。如果不写或传 0 ，这表示仅创建一个 udp handle （用于发送），但不绑定固定端口。

这个函数会返回一个 handle id 。

**socket.udp_connect(id, address, port, callback)**

你可以给一个 udp handle 设置一个默认的发送目的地址。当你用 socket.udp 创建出一个非监听状态的 handle 时，设置目的地址非常有用。因为你很难有别的方法获得一个有效的供 socket.sendto 使用的地址串。这里 callback 是可选项，通常你应该在 socket.udp 创建出 handle 时就设置好 callback 函数。但有时，handle 并不是当前 service 创建而是由别处创建出来的。这种情况，你可以用 socket.start 重设 handle 的所有权，并用这个函数设置 callback 函数。

设置完默认的目的地址后，之后你就可以用 socket.write 来发送数据包。

注：**handle 只能属于一个 service ，当一个 handle 归属一个 service 时，skynet 框架将对应的网络消息转发给它。向一个 handle 发送网络数据包则不需要当前 service 拥有这个 handle 。**

**socket.sendto(id, from, data)**

向一个网络地址发送一个数据包。第二个参数 from 即是一个网络地址，**这是一个 string ，通常由 callback 函数生成，你无法自己构建一个地址串，但你可以把 callback 函数中得到的地址串保存起来以后使用**。发送的内容是一个字符串 data 。

这个字符串可以用 **socket.udp_address(from) : address port** 转换为可读的 ip 地址和端口，用于记录。

------

通过阅读实现代码 lualib/socket.lua ，你可以更清楚的了解 socket api 是如何工作的。你也可能会发现一些并未在上面列出的 api 。它们可能在未来的版本中废弃。

如果你需要一个网关帮你接入大量连接并转发它们到不同的地方处理。service/gate.lua 可以直接使用，同时也是用于了解 skynet 的 socket 模块如何工作的不错的参考。它还有一个功能近似的，但是全部用 C 编写的版本 service-src/service_gate.c 。

如果仅仅是了解 socket api 的基本用法，以及搭建一个简单的服务器。请参考 test/testsocket.lua ；udp 部分见 test/testudp.lua 。

### 域名查询

**在 skynet 的底层，当使用域名而不是 ip 时，由于调用了系统 api getaddrinfo ，有可能阻塞住整个 socket 线程（不仅仅是阻塞当前服务，而是阻塞整个 skynet 节点的网络消息处理）。虽然大多数情况下，我们并不需要向外主动建立连接。但如果你使用了类似 httpc 这样的模块以域名形式向外请求时，一定要关注这个问题。**

skynet 暂时不打算在底层实现非阻塞的域名查询。**但提供了一个上层模块来辅助你解决 dns 查询时造成的线程阻塞问题。**

```
local dns = require "skynet.dns"
```

这样可以加载这个模块，它有两个方法。

在使用前，必须设置 dns 服务器。

- dns.server(p, port) ： port 的默认值为 53 。如果不填写 ip 的话，将从 /etc/resolv.conf 中找到合适的 ip 。
- dns.resolve(name, ipv6) : 查询 name 对应的 ip ，如果 ipv6 为 true 则查询 ipv6 地址，默认为 false 。如果查询失败将抛出异常，成功则返回 ip ，以及一张包含有所有 ip 的 table 。
- dns.flush() : 默认情况下，模块会根据 TTL 值 cache 查询结果。在查询超时的情况下，也可能返回之前的结果。`dns.flush()` 可以用来清空 cache 。注意：cache 保存在调用者的服务中，并非针对整个 skynet 进程。所以，推荐写一个独立的 dns 查询服务统一处理 dns 查询。

你可以阅读 test/testdns.lua 看看范例。同时，这个模块也可以作为在 skynet 中使用 udp 的范例。