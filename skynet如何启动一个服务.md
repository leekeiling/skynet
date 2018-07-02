## skynet如何启动一个服务

#### lua部分

首先看一个skynet_sample中启动agent服务的例子：

 manager.lua

```lua
local function new_agent()
	return skynet.newservice "agent"
end
```

skynet.newservice

skynet.lua

```lua
function skynet.newservice(name, ...)
	return skynet.call(".launcher", "lua" , "LAUNCH", "snlua", name, ...)
end
```

.launcher是一个服务的名字，后面都是通过skynet.call传给.launcher的参数。

我们先来看看这个.launcher是个什么鬼。

bootstrap.lua

```lua
local launcher = assert(skynet.launch("snlua","launcher"))
skynet.name(".launcher", launcher)
```

bootstrap.lua是skynet服务的启动入口，在这里，调用了skynet.launch，启动了一个launcher服务，并将其命名为.launcher。

manager.lua

```lua
local c = require "skynet.core"
function skynet.launch(...)
	local addr = c.command("LAUNCH", table.concat({...}," "))
	if addr then
		return tonumber("0x" .. string.sub(addr , 2))
	end
end
```

这里的skynet.core是一个C语言模块，至此，我们将进入C语言实现部分，调用skynet.core.command(“LAUNCH”, “snlua launcher”)。不要晕，我们先总结一下lua部分的内容：

newservice–>skynet.call .launcher–>.launcher=skynet.launch(“snlua”, “launcher”)–>skynet.core.command(“LAUNCH”, “snlua launcher”)

接下来我们看看skynet.core中是如何实现的。

#### C部分

skynet.core其实是在lua_skynet.c中定义的，其command对应于lcommand函数。 这时的参数其实都压进了lua_State中。

lua-skynet.c

```c++
static int
lcommand(lua_State *L) {
	struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1));
	const char * cmd = luaL_checkstring(L,1);
	const char * result;
	const char * parm = NULL;
	if (lua_gettop(L) == 2) {
		parm = luaL_checkstring(L,2);
	}

	result = skynet_command(context, cmd, parm);
	if (result) {
		lua_pushstring(L, result);
		return 1;
	}
	return 0;
}
```

所以，这里会转发到cmd_launch函数。

skynet_server.c

```c++
static const char *
cmd_launch(struct skynet_context * context, const char * param) {
	size_t sz = strlen(param);
	char tmp[sz+1];
	strcpy(tmp,param);
	char * args = tmp;
	char * mod = strsep(&args, " \t\r\n");
	args = strsep(&args, "\r\n");
	struct skynet_context * inst = skynet_context_new(mod,args);
	if (inst == NULL) {
		return NULL;
	} else {
		id_to_hex(context->result, inst->handle);
		return context->result;//返回服务handle
	}
}
```

在cmd_launch中，mod是snlua，args是“snlua launcher”，会根据这两个参数，重新构造一个skynet_context出来。

skynet_context_new的函数体比较长，其中重要的步骤包括：

1. 根据参数实例化一个模块（这里是snlua)
2. 创建此服务的消息队列
3. 根据参数，初始化前面创建的模块（snlua)

在第1步中，加载模块（snlua)并调用了模块的create函数。

service_snlua.c

```
struct snlua *
snlua_create(void) {
	struct snlua * l = skynet_malloc(sizeof(*l));
	memset(l,0,sizeof(*l));
	l->mem_report = MEMORY_WARNING_REPORT;
	l->mem_limit = 0;
	l->L = lua_newstate(lalloc, l);
	return l;
}
```

这里，新创建了一个lua_State。因此，正如wiki中所说，snlua是lua的一个沙盒服务，保证了各个lua服务之间是隔离的。

而第3步，其实是调用了snlua模块的init函数。

service_snlua.c

```c++
int
snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {
	int sz = strlen(args);
	char * tmp = skynet_malloc(sz);
	memcpy(tmp, args, sz);
	skynet_callback(ctx, l , launch_cb);
	const char * self = skynet_command(ctx, "REG", NULL);
	uint32_t handle_id = strtoul(self+1, NULL, 16);
	// it must be first message
	skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);
	return 0;
}
```

这里，设置了当前模块的callback为launch_cb，因此之后skynet_send消息，将由launch_cb处理。

service_snlua.c

```c++
static int
launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz) {
	assert(type == 0 && session == 0);
	struct snlua *l = ud;
	skynet_callback(context, NULL, NULL);
	int err = init_cb(l, context, msg, sz);
	if (err) {
		skynet_command(context, "EXIT", NULL);
	}

	return 0;
}
```

这里，launch_cb重置了服务的callback（因为snlua只用负责在沙盒中启动lua服务，其他消息应由lua程序处理），之后，调用了init_cb。

init_cb中除了设置各种路径、栈数据之外，和我们关心的lua程序有关的，是这样的一行：

service_snlua.c

```c++
const char * loader = optstring(ctx, "lualoader", "./lualib/loader.lua");

int r = luaL_loadfile(L,loader);
if (r != LUA_OK) {
	skynet_error(ctx, "Can't load %s : %s", loader, lua_tostring(L, -1));
	report_launcher_error(ctx);
	return 1;
}
lua_pushlstring(L, args, sz);
r = lua_pcall(L,1,0,1);
```

这里，就又通过C语言的lua接口，调用回了lua层面。

C语言层面工作：

1、创建skynet_context

- 实例化snlua模块
- 创建消息队列
- 初始化snlua模块

2、注册回调函数。之后skynet_send消息将由回调函数处理

总结一下，C语言层面的处理流程是这样的：

skynet.core.command–>lcommand–>skynet_command–>cmd_launch–>skynet_context_new–>snlua_create–>snlua_init–>loader.lua

#### 回到lua

loader.lua的功能也很简单，就是在沙盒snlua中，加载并执行lua程序，这里也就是launcher.lua。

在launcher.lua中，通过skynet.register_protocol和skynet.dispatch，设置了launcher服务对各种消息的处理函数。而在skynet.start的调用中：

skynet.lua

```lua
function skynet.start(start_func)
	c.callback(skynet.dispatch_message)
	skynet.timeout(0, function()
		skynet.init_service(start_func)
	end)
end
```

这里又重新设置了服务的callback。所以，所谓启动一个服务，其实就是将用lua编写的若干回调函数，挂载到对消息队列的处理中去。具体到这里的launcher服务，其实就是在lua层面，将command.LAUNCH等lua函数，挂接到消息队列中的LAUNCH等消息的回调函数。

### 还没完

到这里，我们看到了launcher.lua这个服务，也就是.launcher这个服务是如何建立的了。但是最开始，我们要考察的是agent.lua服务啊。回过头来，我们再看看agent服务是如何创建的：

manager.lua

```lua
local function new_agent()
	return skynet.newservice "agent"
end
```

skynet.lua

```lua
function skynet.newservice(name, ...)
	return skynet.call(".launcher", "lua" , "LAUNCH", "snlua", name, ...)
end
```

现在，我们知道.launcher从何而来，那就直接看看LAUNCH消息是如何处理的吧：

launcher.lua

```lua
function command.LAUNCH(_, service, ...)
	launch_service(service, ...)
	return NORET
end
```

在launch_service中，调用到了skynet.launch。在最开始就看到了，.launcher的创建，也是通过调用skynet.launch实现的。这样，通过.launcher创建一个服务，和创建.launcher的服务的流程，就统一起来了。

.launcher是一个用于创建其他lua服务的服务。

### 最初的最初

作为创建服务的服务.launcher，它自己又是被谁创建的呢？前面我们看到，它是在bootstrap.lua中创建出来的。那么bootstrap.lua又是什么时候启动的呢？

这里，我们需要看一下skynet启动时的第一个函数入口，main函数。

main函数隐藏在skynet_main.c中，当其解析完传入的config文件之后，就转到了skynet_start。

在skynet_start函数中，调用了bootstrap(ctx, config->bootstrap),其中，就像前面讲到的流程一样，直接走到了skynet_context_new这一步。

一般默认，config->bootstrap项就是snlua bootstrap。那这里其实就是启动调用bootstrap.lua，，其中会启动launcher服务。有了launcher服务，之后的服务启动，就都可以交由launcher服务来进行了。

### 综述

启动一个服务，调用skynet_call将创建消息发送到launch服务。launch服务处理其他服务的创建消息，负责服务的创建。

launch服务的创建和经过launch服务创建服务流程一致。