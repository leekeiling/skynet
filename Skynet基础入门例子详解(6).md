### Skynet基础入门例子详解(6)

#### 把socket控制权交给其他服务

socket.abandon(id) 清除 socket id 在本服务内的数据结构，但并不关闭这个 socket 。这可以用于你把 id 发送给其它服务，以转交 socket 的控制权。

在同一个目录建立7个文件（config，proto.lua，main.lua，socket1.lua，client1.lua，agent1.lua，agent2.lua）

本节例子与上一节例子主要区别是： 
socket1服务不处理接收到的消息，交给了agent处理

socket1.lua代码：

```lua
local skynet = require "skynet"
require "skynet.manager"    -- import skynet.register
local socket = require "socket"

local function accept1(id)
    socket.start(id)
    socket.write(id, "Hello Skynet\n")
    skynet.newservice("agent1", id)
    -- notice: Some data on this connection(id) may lost before new service start.
    -- So, be careful when you want to use start / abandon / start .
    socket.abandon(id)
end


local function accept2(id)
    socket.start(id)
    local agent2 = skynet.newservice("agent2")
    skynet.call(agent2,"lua",id)

    -- socket.abandon(id)清除 socket id 在本服务内的数据结构，但并不关闭这个 socket 。这可以用于你把 id 发送给其它服务，以转交 socket 的控制权。
    socket.abandon(id)
end

skynet.start(function()
    print("==========Socket Start=========")
    local id = socket.listen("127.0.0.1", 8888)
    print("Listen socket :", "127.0.0.1", 8888)

    socket.start(id , function(id, addr)
            -- 接收到客户端连接或发送消息()
            print("connect from " .. addr .. " " .. id)

            -- 处理接收到的消息（交给angent1或agent2处理）
            -- accept1(id)
            accept2(id)
        end)
    --可以为自己注册一个别名。（别名必须在 32 个字符以内）
    skynet.register "SOCKET1"
end)
```

agent1.lua代码：

```lua
local skynet = require "skynet"
local socket = require "socket"

local fd = ...

fd = tonumber(fd)

local function echo(id)
    socket.start(id)

    while true do
        local str = socket.read(id)
        if str then
            print("client say:"..str)
            -- socket.write(id, str)
        else
            socket.close(id)
            return
        end
    end
end

skynet.start(function()
    skynet.fork(function()
        echo(fd)
        skynet.exit()
    end)
end)
```

agent2.lua代码：

```lua
local skynet = require "skynet"
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

local function echo(id)
    socket.start(id)

    host = sproto.new(proto.c2s):host "package"

    while true do
        local str = socket.read(id)
        if str then
            local type,str2,str3,str4 = host:dispatch(str)

            if type=="REQUEST" then
                local ok, result  = pcall(request, str2,str3,str4)
                -- print("client say:"..str)
            end

            -- socket.write(id, str)
        else
            socket.close(id)
            return
        end
    end
end

skynet.start(function()
    skynet.dispatch("lua", function(session, address, fd, ...)

        skynet.fork(function()
            echo(fd)
            skynet.exit()
        end)

    end)    
end)
```

