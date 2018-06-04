## Skynet基础入门例子详解(3)

## 服务端与客户端的Socket通信

使用Skynet的Socket通信，看官方的例子（example2/client.lua和test/testsocket.lua），不懂sproto协议的同学还真有点懵逼。下面我用我们常用的编程思维来实现一个简单的Socket通信功能，方便大家理解其中的原理。

在同一个目录建立4个文件（config，main.lua，socket1.lua，client1.lua） 
config文件参考上一节

main.lua代码：

```lua
local skynet = require "skynet"

-- 启动服务(启动函数)
skynet.start(function()
    -- 启动函数里调用Skynet API开发各种服务
    print("======Server start=======")

    skynet.newservice("socket1")
    skynet.exit()
end)
```

socket1.lua代码：

```lua
local skynet = require "skynet"
local socket = require "socket"

-- 读取客户端数据, 并输出
local function echo(id)
    -- 每当 accept 函数获得一个新的 socket id 后，并不会立即收到这个 socket 上的数据。这是因为，我们有时会希望把这个 socket 的操作权转让给别的服务去处理。
    -- 任何一个服务只有在调用 socket.start(id) 之后，才可以收到这个 socket 上的数据。
    socket.start(id)

    while true do
        -- 读取客户端发过来的数据
        local str = socket.read(id)
        if str then
            -- 直接打印接收到的数据
            print(str)
        else
            socket.close(id)
            return
        end
    end
end

skynet.start(function()
    print("==========Socket1 Start=========")
    -- 监听一个端口，返回一个 id ，供 start 使用。
    local id = socket.listen("127.0.0.1", 8888)
    print("Listen socket :", "127.0.0.1", 8888)

    socket.start(id , function(id, addr)
            -- 接收到客户端连接或发送消息()
            print("connect from " .. addr .. " " .. id)

            -- 处理接收到的消息
            echo(id)

        end)
    --可以为自己注册一个别名。（别名必须在 32 个字符以内）
    -- skynet.register "SOCKET1"
end)
```

```lua
package.cpath = "luaclib/?.so"
package.path = "lualib/?.lua;myexample/e1/?.lua"

if _VERSION ~= "Lua 5.3" then
    error "Use lua 5.3"
end

local socket = require "clientsocket"

local fd = assert(socket.connect("127.0.0.1", 8888))

-- 发送一条消息给服务器, 消息协议可自定义(官方推荐sproto协议,当然你也可以使用最熟悉的json)
socket.send(fd, "Hello world")

```

