## Skynet基础入门例子详解(5)

#### Socket通信协议Sproto

在和客户端通讯时，需要制订一套通讯协议。 skynet 并没有规定任何通讯协议，所以你可以自由选择。

sproto 是一套由 skynet 自身提供的协议，并没有特别推荐使用，只是一个选项。sproto 有一个独立项目存在 。同时也复制了一份在 skynet 的源码库中。

在同一个目录建立5个文件（config，proto.lua，main.lua，socket1.lua，client1.lua） 
config文件参考第一节内容

proto.lua是定义通信协议，代码：

```lua
local sprotoparser = require "sprotoparser"

local proto = {}

proto.c2s = sprotoparser.parse [[
.package {
    type 0 : integer
    session 1 : integer
}

handshake 1 {
    response {
        msg 0  : string
    }
}

say 2 {
    request {
        name 0 : string
        msg 1 : string
    }
}

quit 3 {}

]]

proto.s2c = sprotoparser.parse [[
.package {
    type 0 : integer
    session 1 : integer
}

heartbeat 1 {}
]]

return proto

```

main.lua代码：

```lua
local skynet = require "skynet"

-- 启动服务(启动函数)
skynet.start(function()
    -- 启动函数里调用Skynet API开发各种服务
    print("======Server start=======")
    -- skynet.newservice(name, ...)启动一个新的 Lua 服务(服务脚本文件名)

    skynet.newservice("socket1")

    -- 退出当前的服务
    skynet.exit()
end)

```

socket1.lua服务端：

```lua
local skynet = require "skynet"
require "skynet.manager"    -- import skynet.register
local socket = require "socket"

local proto = require "proto"
local sproto = require "sproto"

local host

local REQUEST = {}

function REQUEST:say()
    print("say", self.name, self.msg)
end

function REQUEST:handshake()
    print("handshake")
end

function REQUEST:quit()
    print("quit")
end

local function request(name, args, response)
    local f = assert(REQUEST[name])
    local r = f(args)
    if response then
        -- 生成回应包(response是一个用于生成回应包的函数。)
        -- 处理session对应问题
        -- return response(r)
    end
end

local function send_package(fd,pack)
    -- 协议与客户端对应(两字节长度包头+内容)
    local package = string.pack(">s2", pack)
    socket.write(fd, package)
end

local function accept(id)
    -- 每当 accept 函数获得一个新的 socket id 后，并不会立即收到这个 socket 上的数据。这是因为，我们有时会希望把这个 socket 的操作权转让给别的服务去处理。
    -- 任何一个服务只有在调用 socket.start(id) 之后，才可以收到这个 socket 上的数据。

    socket.start(id)

    host = sproto.new(proto.c2s):host "package"
    -- request = host:attach(sproto.new(proto.c2s))

    while true do
        local str = socket.read(id)
        if str then
            local type,str2,str3,str4 = host:dispatch(str)

            if type=="REQUEST" then
                -- REQUEST : 第一个返回值为 "REQUEST" 时，表示这是一个远程请求。如果请求包中没有 session 字段，表示该请求不需要回应。这时，第 2 和第 3 个返回值分别为消息类型名（即在 sproto 定义中提到的某个以 . 开头的类型名），以及消息内容（通常是一个 table ）；如果请求包中有 session 字段，那么还会有第 4 个返回值：一个用于生成回应包的函数。
                local ok, result  = pcall(request, str2,str3,str4)

                if ok then
                    if result then
                        socket.write(id, "收到了")
                        -- 暂时不使用回应包回应
                        -- print("response:"..result)
                        -- send_package(id,result)
                    end
                else
                    skynet.error(result)
                end
            end

            if type=="RESPONSE" then
                -- RESPONSE ：第一个返回值为 "RESPONSE" 时，第 2 和 第 3 个返回值分别为 session 和消息内容。消息内容通常是一个 table ，但也可能不存在内容（仅仅是一个回应确认）。
                -- 暂时不处理客户端的回应
                print("client response")
            end         

        else
            socket.close(id)
            return
        end
    end
end

skynet.start(function()
    print("==========Socket Start=========")
    local id = socket.listen("127.0.0.1", 8888)
    print("Listen socket :", "127.0.0.1", 8888)

    socket.start(id , function(id, addr)
            -- 接收到客户端连接或发送消息()
            print("connect from " .. addr .. " " .. id)

            -- 处理接收到的消息
            accept(id)
        end)
    --可以为自己注册一个别名。（别名必须在 32 个字符以内）
    skynet.register "SOCKET1"
end)
```

client1.lua客户端：

```lua
package.cpath = "luaclib/?.so"
package.path = "lualib/?.lua;myexample/e5/?.lua"

if _VERSION ~= "Lua 5.3" then
    error "Use lua 5.3"
end

local socket = require "clientsocket"

-- 通信协议
local proto = require "proto"
local sproto = require "sproto"

local host = sproto.new(proto.s2c):host "package"
local request = host:attach(sproto.new(proto.c2s))

local fd = assert(socket.connect("127.0.0.1", 8888))

local session = 0
local function send_request(name, args)
    session = session + 1
    local str = request(name, args, session)

    -- 解包测试
    -- local host2 = sproto.new(proto.c2s):host "package"
    -- local type,str2 = host2:dispatch(str)
    -- print(type)
    -- print(str2)

    socket.send(fd, str)

    print("Request:", session)
end

send_request("handshake")
send_request("say", { name = "soul", msg = "hello world" })

while true do
    -- 接收服务器返回消息
    local str   = socket.recv(fd)

    -- print(str)
    if str~=nil and str~="" then
            print("server says: "..str)
            -- socket.close(fd)
            -- break;
    end

    -- 读取用户输入消息
    local readstr = socket.readstdin()

    if readstr then
        if readstr == "quit" then
            send_request("quit")
            -- socket.close(fd)
            -- break;
        else
            -- 把用户输入消息发送给服务器
            send_request("say", { name = "soul", msg = readstr })
        end
    else
        socket.usleep(100)
    end

end

```

