## Bootstrap

skynet 由一个或多个进程构成，每个进程被称为一个 skynet 节点。本文描述了 skynet 节点的启动流程。

skynet 节点通过运行 skynet 主程序启动，必须在启动命令行传入一个 [Config](https://github.com/cloudwu/skynet/wiki/Config) 文件名作为启动参数。skynet 会读取这个 config 文件获得启动需要的参数。

**第一个启动的服务是 logger ，它负责记录之后的服务中的 log 输出**。logger 是一个简单的 C 服务，`skynet_error` 这个 C API 会把字符串发送给它。在 config 文件中，logger 配置项可以配置 log 输出的文件名，默认是 nil ，表示输出到标准输出。

**bootstrap 这个配置项关系着 skynet 运行的第二个服务。通常通过这个服务把整个系统启动起来。**默认的 bootstrap 配置项为 `"snlua bootstrap"` ，这意味着，skynet 会启动 snlua 这个服务，并将 bootstrap 作为参数传给它。snlua 是 lua 沙盒服务，bootstrap 会根据配置的 luaservice 匹配到最终的 lua 脚本。如果按默认配置，**这个脚本应该是 service/bootstrap.lua** 。

**如无必要，你不需要更改 bootstrap 配置项，让默认的 bootstrap 脚本工作**。目前的 bootstrap 脚本如下：

```
local skynet = require "skynet"
local harbor = require "skynet.harbor"

skynet.start(function()
	local standalone = skynet.getenv "standalone"

	local launcher = assert(skynet.launch("snlua","launcher"))
	skynet.name(".launcher", launcher)

	local harbor_id = tonumber(skynet.getenv "harbor")
	if harbor_id == 0 then
		assert(standalone ==  nil)
		standalone = true
		skynet.setenv("standalone", "true")

		local ok, slave = pcall(skynet.newservice, "cdummy")
		if not ok then
			skynet.abort()
		end
		skynet.name(".slave", slave)

	else
		if standalone then
			if not pcall(skynet.newservice,"cmaster") then
				skynet.abort()
			end
		end

		local ok, slave = pcall(skynet.newservice, "cslave")
		if not ok then
			skynet.abort()
		end
		skynet.name(".slave", slave)
	end

	if standalone then
		local datacenter = skynet.newservice "datacenterd"
		skynet.name("DATACENTER", datacenter)
	end
	skynet.newservice "service_mgr"
	pcall(skynet.newservice,skynet.getenv "start" or "main")
	skynet.exit()
end)
```

**这段脚本通常会根据 standalone 配置项判断你启动的是一个 master 节点还是 slave 节点。如果是 master 节点还会进一步的通过 harbor 是否配置为 0 来判断你是否启动的是一个单节点 skynet 网络。**

单节点模式下，是不需要通过内置的 harbor 机制做节点间通讯的。但为了兼容（因为你还是有可能注册全局名字），需要启动一个叫做 cdummy 的服务，它负责拦截对外广播的全局名字变更。

如果是多节点模式，对于 master 节点，需要启动 cmaster 服务作节点调度用。此外，每个节点（包括 master 节点自己）都需要启动 cslave 服务，用于节点间的消息转发，以及同步全局名字。

接下来在 master 节点上，还需要启动 [DataCenter](https://github.com/cloudwu/skynet/wiki/DataCenter) 服务。

然后，启动用于 [UniqueService](https://github.com/cloudwu/skynet/wiki/UniqueService) 管理的 `service_mgr` 。

**最后，它从 config 中读取 start 这个配置项，作为用户定义的服务启动入口脚本运行。成功后，把自己退出。**

**这个 start 配置项，才是用户定义的启动脚本，默认值为 "main" 。**如果你只是试玩一下 skynet ，可能有多份不同的启动脚本，那么建议你多写几份 config 文件，在里面配置不同的 start 项。examples 目录下有很多这样的例子。