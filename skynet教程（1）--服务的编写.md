## skynet教程（1）--服务的编写

### 一、首先引入框架 

`local skynet = require "skynet"`

然后要准备一个回调函数，每个服务都有一个回调函数，这个回调函数是被skynet框架调用的，当有消息投递到服务上时，skynet框架就会调用服务的回调函数对消息进行处理。这个回调函数是skynet.dispatch。

最后要使用skynet.start把服务启动起来。

skyet/example/echo.lua

```c++
local skynet = require "skynet"
require "skynet.manager"


local command = {}

function command.HELLO(what)
    return "i am echo server, get this:" .. what
end

function dispatcher() 
    skynet.dispatch("lua", function(session, address, cmd, ...)
        cmd = cmd:upper()
        if cmd == "HELLO" then
            local f = command[cmd]
            assert(f)
            skynet.ret(skynet.pack(f(...)))
        end
    end)
    skynet.register("echo")
end

skynet.start(dispatcher)
```

二、写完上面的代码是不是就大功告成了呢，实际上并没有。要测这个服务，我们还需要另写一个服务，可能有的人心中默默地已经在念三字经了。在没有搭建服务端，纯在skynet环境中运行测试，目前只能这样了

skynet/example/test_echo.lua

```lua
local skynet = require "skynet"

skynet.start(function() 
    local echo = skynet.newservice("echo")
    print(skynet.call(echo, "lua", "HELLO", "world"))
end);
```

注意不要直接使用skynet去启动test_echo.lua脚本，一定会报错的。不管我这里怎么讲，我相信一定会有人要去试一试的。

skynet.newservice是启动echo服务。

skynet.call是调用服务，其中参数echo就是上面dispatch对应的address，参数”lua”就是dispatch中的lua，”HELLO”就是上面对应的cmd。参数”world”就是…里面的内容了。call是阻塞式调用。

到了这一步是不是就可以了？不不不，还有第三步，不过第三步很简单。

三、修改配置文件example/config 
在源码分析中分析过了，skynet一定需要一个配置文件，在里面配置cpath/thread之类的信息。先备份一下example/config文件。然后再个性example/config文件，把start那一行

四、好了，现在运行 
`./skynet ./example/config`