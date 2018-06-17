## skynet如何启动一个c服务

我们写的c服务在编译成so库以后，在某个时段，会被加载到modules列表中。创建c服务的工作，一般在c层进行，一般会调用skynet_context_new接口，如下所示：

```c++
// skynet_server.c
struct skynet_context * 
skynet_context_new(const char * name, const char *param) {
    struct skynet_module * mod = skynet_module_query(name);

    if (mod == NULL)
        return NULL;

    void *inst = skynet_module_instance_create(mod);
    if (inst == NULL)
        return NULL;
    struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
    CHECKCALLING_INIT(ctx)

    //初始化
    ctx->mod = mod;
    ctx->instance = inst; //一般是context_module实例
    ctx->ref = 2;
    ctx->cb = NULL;
    ctx->cb_ud = NULL;
    ctx->session_id = 0;
    ctx->logfile = NULL;

    ctx->init = false;
    ctx->endless = false;
    // Should set to 0 first to avoid skynet_handle_register get an uninitialized handle
    ctx->handle = 0;    
    ctx->handle = skynet_handle_register(ctx);
    struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle);
    // init function maybe use ctx->handle, so it must init at last
    context_inc();

    CHECKCALLING_BEGIN(ctx)
    
    int r = skynet_module_instance_init(mod, inst, ctx, param);
    CHECKCALLING_END(ctx)
    if (r == 0) {
        struct skynet_context * ret = skynet_context_release(ctx);
        if (ret) {
            ctx->init = true;
        }
        skynet_globalmq_push(queue);
        if (ret) {
            skynet_error(ret, "LAUNCH %s %s", name, param ? param : "");
        }
        return ret;
    } else {
        skynet_error(ctx, "FAILED launch %s", name);
        uint32_t handle = ctx->handle;
        skynet_context_release(ctx);
        skynet_handle_retire(handle);
        struct drop_t d = { handle };
        skynet_mq_release(queue, drop_message, &d);
        return NULL;
    }
}

// skynet_module.c
struct skynet_module * 
skynet_module_query(const char * name) {
    struct skynet_module * result = _query(name);
    if (result)
        return result;

    SPIN_LOCK(M)

    result = _query(name); // double check

    if (result == NULL && M->count < MAX_MODULE_TYPE) {
        int index = M->count;
        void * dl = _try_open(M,name);
        if (dl) {
            M->m[index].name = name;
            M->m[index].module = dl;

            if (_open_sym(&M->m[index]) == 0) {
                M->m[index].name = skynet_strdup(name);
                M->count ++;
                result = &M->m[index];
            }
        }
    }

    SPIN_UNLOCK(M)

    return result;
}

static void *
_try_open(struct modules *m, const char * name) {
    const char *l;
    const char * path = m->path;
    size_t path_size = strlen(path);
    size_t name_size = strlen(name);

    int sz = path_size + name_size;
    //search path
    void * dl = NULL;
    char tmp[sz];
    do
    {
        memset(tmp,0,sz);
        while (*path == ';') path++;
        if (*path == '\0') break;
        l = strchr(path, ';');
        if (l == NULL) l = path + strlen(path);
        int len = l - path;
        int i;
        for (i=0;path[i]!='?' && i < len ;i++) {
            tmp[i] = path[i];
        }
        memcpy(tmp+i,name,name_size);
        if (path[i] == '?') {
            strncpy(tmp+i+name_size,path+i+1,len - i - 1);
        } else {
            fprintf(stderr,"Invalid C service path\n");
            exit(1);
        }
        dl = dlopen(tmp, RTLD_NOW | RTLD_GLOBAL);
        path = l;
    }while(dl == NULL);

    if (dl == NULL) {
        fprintf(stderr, "try open %s failed : %s\n",name,dlerror());
    }

    return dl;
}

_open_sym(struct skynet_module *mod) {
    size_t name_size = strlen(mod->name);
    char tmp[name_size + 9]; // create/init/release/signal , longest name is release (7)
    memcpy(tmp, mod->name, name_size);
    strcpy(tmp+name_size, "_create");
    mod->create = dlsym(mod->module, tmp);
    strcpy(tmp+name_size, "_init");
    mod->init = dlsym(mod->module, tmp);
    strcpy(tmp+name_size, "_release");
    mod->release = dlsym(mod->module, tmp);
    strcpy(tmp+name_size, "_signal");
    mod->signal = dlsym(mod->module, tmp);

    return mod->init == NULL;
}
```

我们要创建一个c服务，首先要获取对应c服务的模块对象，在上一节中，我们已经介绍了skynet的modules对象，它包含skynet_module列表，so库所在路径，创建一个c服务，一般要经历下面几个步骤：

1. 从modules列表中，查找对应的服务模块，如果找到则返回，否则到modules的path中去查找对应的so库，创建一个skynet_module对象（数据结构见上节），将so库加载到内存，并将访问该so库的句柄和skynet_module对象关联（_try_open做了这件事），并将so库中的xxx_create，xxx_init，xxx_signal，xxx_release四个函数地址赋值给skynet_module的create、init、signal和release四个函数中，这样这个skynet_module对象，就能调用so库中，对应的四个接口（_open_sym做了这件事）。**创建skynet_module对象，加载so到内存，so库句柄与skynetmodule对象关联。创建module对象**
2. 创建一个服务实例即skynet_context对象，他包含一个次级消息队列指针，服务模块指针（skynet_module对象，便于他访问module自定义的create、init、signal和release函数），由服务模块调用create接口创建的数据实例等。**创建skynet_context对象，关联skynet_module**
3. 将新创建的服务实例（skynet_context对象）注册到全局的服务列表中。**将skynet_context注册到全局服务列表中。**
4. 初始化服务模块（skynet_module创建的数据实例），并在初始化函数中，注册新创建的skynet_context实例的callback函数。**初始化和注册回调函数。**
5. 将该服务实例（skynet_context实例）的次级消息队列，插入到全局消息队列中。**将次级消息队列插入到全局消息队列中。**
   经过上面的步骤，一个c服务模块就被创建出来了，在回调函数被指定以后，其他服务发送给他的消息，会被pop出来，最终传给服务对应的callback函数，最后达到驱动服务的目的。

下面通过创建logger服务的例子，来进行说明：

1. 启动skynet节点时，会启动一个logger c服务

   ```c++
   // skynet_start.c
   void
   skynet_start(struct skynet_config * config) {
   ...
   struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);
   if (ctx == NULL) {
       fprintf(stderr, "Can't launch %s service\n", config->logservice);
       exit(1);
   }
   ...
   }
   ```

   其中配置中的logservice指名log服务的so库名称，用于后面加载服务时使用，后面的logger字段则指定log的输出路径。

2. 此时，在skynet_module列表中，搜索logger服务模块，如果没找到则在so库的输出路径中，寻找名为logger的so库，找到则将该so库加载到内存中，并将对应的logger_create,logger_init,logger_release函数地址分别赋值给logger模块中的create,init,release函数指针，此时图1中的skynet_module列表中，多了一个logger模块。

3. 创建服务实例，即创建一个skynet_context实例，为了使skynet_context实例拥有访问logger服务内部函数的权限，这里将logger模块指针，赋值给skynet_context实例的mod变量中。

4. 创建一个logger服务的数据实例，调用logger服务的create函数：

   ```c++
   // service_logger.c
   struct logger { //服务的数据实例
   FILE * handle;
   int close;
   };
   struct logger *
   logger_create(void) { //
   struct logger * inst = skynet_malloc(sizeof(*inst));
   inst->handle = NULL;
   inst->close = 0;
   return inst;
   }
   ```

   此时，将新创建的数据实例赋值给skynet_context的instance变量，此时，一个服务对象运行时，所要用到的逻辑，能够通过mod变量，访问logger服务对应的函数，而通过instance可以找到该服务自己的数据块。

5. 将新创建的skynet_context对象，注册到图一的skynet_context list中，此时skynet_context list多了一个logger服务实例

6. 初始化logger服务，注册logger服务的callback函数：

   ```c++
   // service_logger.c
   static int
   _logger(struct skynet_context * context, void *ud, int type, int session, uint32_t source, const void * msg, size_t sz) {
   struct logger * inst = ud;
   fprintf(inst->handle, "[:%08x] ",source);
   fwrite(msg, sz , 1, inst->handle);
   fprintf(inst->handle, "\n");
   fflush(inst->handle);
   return 0;
   }
   int
   logger_init(struct logger * inst, struct skynet_context *ctx, const char * parm) {
   if (parm) {
       inst->handle = fopen(parm,"w");
       if (inst->handle == NULL) {
           return 1;
       }
       inst->close = 1;
   } else {
       inst->handle = stdout;
   }
   if (inst->handle) {
       skynet_callback(ctx, inst, _logger);
       skynet_command(ctx, "REG", ".logger");
       return 0;
   }
   return 1;
   }
   // skynet_server.c
   void 
   skynet_callback(struct skynet_context * context, void *ud, skynet_cb cb) {
   context->cb = cb;
   context->cb_ud = ud;
   }
   ```

   上面这段逻辑，将skynet_context的callback函数设置为logger服务的_logger函数，并将调用callback时，传入的userdata设置为先前创建的数据实例

7. 为logger服务实例创建一个次级消息队列，并将队列插入到全局消息队列中

从上面的例子，我们就完成了一个logger服务的创建了，当logger服务收到消息时，就会调用_logger函数来进行处理，并将日志输出