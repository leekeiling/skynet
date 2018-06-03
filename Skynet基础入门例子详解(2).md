## Skynet基础入门例子详解(2)

同样在同一个目录建立3个文件（config，main.lua，service2.lua） 
config文件参考上一节

main.lua代码：

```lua
local skynet = require "skynet"

-- 启动服务(启动函数)
skynet.start(function()
    -- 启动函数里调用Skynet API开发各种服务
    print("======Server start=======")
    -- skynet.newservice(name, ...)启动一个新的 Lua 服务(服务脚本文件名)

    local service2 = skynet.newservice("service2")
    -- 向service2服务发出请求，设置key1的值
    skynet.call(service2,"lua","set","key1","value111111")  
    -- 向service2服务发出请求，获取key1的值
    local kv = skynet.call(service2,"lua","get","key1")
    print(kv)

    -- 退出当前的服务
    skynet.exit()
end)
```

skynet.send(address, typename, …) 把一条类别为 typename 的消息发送给 address 。它会先经过事先注册的 pack 函数打包 … 的内容。 
skynet.send 是一条非阻塞 API ，发送完消息后，coroutine 会继续向下运行，这期间服务不会重入。

skynet.call(address, typename, …) 这条 API 则不同，它会在内部生成一个唯一 session ，并向 address 提起请求，并阻塞等待对 session 的回应（可以不由 address 回应）。当消息回应后，还会通过之前注册的 unpack 函数解包。

尤其需要留意的是，skynet.call 仅仅阻塞住当前的 coroutine ，而没有阻塞整个服务。在等待回应期间，服务照样可以响应其他请求。所以，尤其要注意，在 skynet.call 之前获得的服务内的状态，到返回后，很有可能改变。

service2.lua代码：

```lua
-- 每个服务独立, 都需要引入skynet
local skynet = require "skynet"
require "skynet.manager"    -- 引入 skynet.register
local db = {}

local command = {}

function command.get(key)
    print("comman.get:"..key)   
    return db[key]
end

function command.set(key, value)
    print("comman.set:key="..key..",value:"..value) 
    db[key] = value
    local last = db[key]
    return last
end

skynet.start(function()
    print("==========Service2 Start=========")
    skynet.dispatch("lua", function(session, address, cmd, ...)
        print("==========Service2 dispatch============"..cmd)
        local f = command[cmd]      
        if f then
            -- 回应一个消息可以使用 skynet.ret(message, size) 。
            -- 它会将 message size 对应的消息附上当前消息的 session ，以及 skynet.PTYPE_RESPONSE 这个类别，发送给当前消息的来源 source 
            skynet.ret(skynet.pack(f(...))) --回应消息
        else
            error(string.format("Unknown command %s", tostring(cmd)))
        end
    end)
    --可以为自己注册一个别名。（别名必须在 32 个字符以内）
    skynet.register "SERVICE2"
end)
```

回应一个消息可以使用 skynet.ret(message, size) 。它会将 message size 对应的消息附上当前消息的 session ，以及 skynet.PTYPE_RESPONSE 这个类别，发送给当前消息的来源 source 。 
由于某些历史原因（早期的 skynet 默认消息类别是文本，而没有经过特殊编码），这个 API 被设计成传递一个 C 指针和长度，而不是经过当前消息的 pack 函数打包。或者你也可以省略 size 而传入一个字符串。

由于 skynet 中最常用的消息类别是 lua ，这种消息是经过 skynet.pack 打包的，所以惯用法是 skynet.ret(skynet.pack(…)) 。 
skynet.pack(…) 返回一个 lightuserdata 和一个长度，符合 skynet.ret 的参数需求；与之对应的是 skynet.unpack(message, size) 它可以把一个 C 指针加长度的消息解码成一组 Lua 对象。

**运行程序：**

```lua
./skynet ./myexample/e1/config
```