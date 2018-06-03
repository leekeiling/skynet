## Skynet基础入门例子详解(1)

#### 最简单的入门例子：

同目录创建3个文件（config，main.lua，service1.lua） 
我这里是skynet安装目录下的：myexample/e1/

config配置（examples例子里面的照抄，修改一下目录）

```lua
root = "./"
thread = 8
logger = nil
logpath = "."
harbor = 1
address = "127.0.0.1:2526"
master = "127.0.0.1:2013"
start = "main"  -- main script
bootstrap = "snlua bootstrap"   -- 启动的第一个服务以及其启动参数 service/bootstrap.lua
standalone = "0.0.0.0:2013"
luaservice = root.."service/?.lua;"..root.."myexample/e1/?.lua"
lualoader = root .. "lualib/loader.lua"
lua_path = root.."lualib/?.lua;"..root.."lualib/?/init.lua"
lua_cpath = root .. "luaclib/?.so"
-- preload = "./example1/preload.lua"   -- run preload.lua before every lua service run
snax = root.."example1/?.lua;"..root.."test/?.lua"
-- snax_interface_g = "snax_g"
cpath = root.."cservice/?.so"
-- daemon = "./skynet.pid"
```

main.lua代码：

```lua
local skynet = require "skynet"

-- 启动服务(启动函数)
skynet.start(function()
    -- 启动函数里调用Skynet API开发各种服务
    print("======Server start=======")
    -- skynet.newservice(name, ...)启动一个新的 Lua 服务(服务脚本文件名)
    skynet.newservice("service1")

    -- 退出当前的服务
    -- skynet.exit 之后的代码都不会被运行。而且，当前服务被阻塞住的 coroutine 也会立刻中断退出。
    skynet.exit()
end)
```

service1.lua代码：

```lua
-- 每个服务独立, 都需要引入skynet
local skynet = require "skynet"

-- 这里可以编写各种服务处理函数

skynet.start(function()
        print("==========Service1 Start=========")
        -- 这里可以编写服务代码，使用skynet.dispatch消息分发到各个服务处理函数（后续例子再说）
end)
```

**代码讲解：** 
从这个例子可以看出skynet的基本工作原理 
skynet使用newservice创建各种独立的服务，这就是云风大神提到的沙盒。 
为每个服务创建沙盒，各个服务独立运行，互不影响。 
各个服务之间可以相互调用，调用方法后面再说。