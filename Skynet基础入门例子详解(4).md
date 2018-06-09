## Skynet基础入门例子详解(4)

#### 服务端与客户端的Socket通信2

在同一个目录建立4个文件（config，main.lua，socket2.lua，client2.lua） 
config文件参考上一节

main.lua代码：

```lua
local skynet = require "skynet"

-- 启动服务(启动函数)
skynet.start(function()
    -- 启动函数里调用Skynet API开发各种服务
    print("======Server start=======")

    skynet.newservice("socket2")
    skynet.exit()
end)
```

socket2.lua代码：

```lua
local skynet = require "skynet"
local socket = require "socket"

local function echo(id)
    -- 每当 accept 函数获得一个新的 socket id 后，并不会立即收到这个 socket 上的数据。这是因为，我们有时会希望把这个 socket 的操作权转让给别的服务去处理。
    -- 任何一个服务只有在调用 socket.start(id) 之后，才可以收到这个 socket 上的数据。
    socket.start(id)

    while true do
        local str = socket.read(id)
        if str then
            print("client say:"..str)
            -- 把一个字符串置入正常的写队列，skynet 框架会在 socket 可写时发送它。
            socket.write(id, str)
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
    -- skynet.register "SOCKET2"
end)
```

client2.lua代码：

```lua
package.cpath = "luaclib/?.so"
package.path = "lualib/?.lua;myexample/e1/?.lua"

if _VERSION ~= "Lua 5.3" then
    error "Use lua 5.3"
end

local socket = require "clientsocket"

local fd = assert(socket.connect("127.0.0.1", 8888))

socket.send(fd, "Hello world")
while true do
    -- 接收服务器返回消息
    local str   = socket.recv(fd)
    if str~=nil and str~="" then
            print("server says: "..str)
            -- socket.close(fd)
            -- break;
    end

    -- 读取用户输入消息
    local readstr = socket.readstdin()
    if readstr then
        if readstr == "quit" then
            socket.close(fd)
            break;
        else
            -- 把用户输入消息发送给服务器
            socket.send(fd, readstr)
        end
    else
        socket.usleep(100)
    end
end
```

通过以上例子，详细大家已基本理解Skynet的Socket通信原理。 
下一节将讲述通信协议的使用。