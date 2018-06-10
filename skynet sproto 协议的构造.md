## skynet sproto 协议的构造

parse 生成的bin串后，用 sproto.new(pbin) 构造协议

```lua
local sproto_mt = { __index = sproto }
function sproto.new(bin)
    local cobj = assert(core.newproto(bin))
    local self = {
        __cobj = cobj,
        __tcache = setmetatable( {} , weak_mt ),
        __pcache = setmetatable( {} , weak_mt ),
    }
    return setmetatable(self, sproto_mt)
end
```

sproto_mt 的 index域被设置为 sproto ， new返回的对象就可以调用sproto的函数了。 

__tcache 用来换成上篇中用户定义的结构体， 相同的。

__pcache 来缓存上篇中定义的 protocol。 

__cobj 则是 sproto c模块生成的 struct sproto结构的userdata。

sproto c结构如下

```c++
struct sproto {
    struct pool memory; //貌似管类内存的指针， 有大神看到帮解释下
    int type_n; //有多少个定义的结构 上篇中"type"元素的个数
    int protocol_n;//协议的个数 上篇中"protocol"元素的个数
    struct sproto_type * type; //结构体结构数组
    struct protocol * proto; //协议结构数组
};

struct sproto_type {
    const char * name; //结构名字
    int n; //字段的个数
    int base;  //
    int maxn; //字段的个数
    struct field *f; //字段信息
};

struct protocol {
    const char *name;// 协议名字
    int tag;
    struct sproto_type * p[2];
};

struct field {
    int tag;  //序号
    int type; //类型
    const char * name; //名字
    struct sproto_type * st; //自定义类型的话， 这里是指向自定义类型的指针
    int key;
};
```

在new 中 调用了 sproto.c 的 struct sproto * sproto_create(const void * proto, size_t sz)函数 创建协议的c结构。 
const void * proto 是new 传入的bin串 sz 是bin串大小。

```c++
struct sproto *
sproto_create(const void * proto, size_t sz) {
    struct pool mem;
    struct sproto * s;
    pool_init(&mem);
    s = pool_alloc(&mem, sizeof(*s));
    if (s == NULL)
        return NULL;
    memset(s, 0, sizeof(*s));
    s->memory = mem;
    if (create_from_bundle(s, proto, sz) == NULL) {
        pool_release(&s->memory);
        return NULL;
    }
    return s;
}
```

在 static struct sproto * create_from_bundle(struct sproto *s, const uint8_t *stream, size_t sz) 

之前都是初始化 sproto c结构体， create_from_bundle,成功了就返回 sproto* 失败返回NULL. 返回的 sproto* 就是lua里面的 __cobj 了， 我们进入 create_from_bundle 继续走

```c++
static struct sproto *
create_from_bundle(struct sproto *s, const uint8_t * stream, size_t sz) {
    const uint8_t * content;
    const uint8_t * typedata = NULL;
    const uint8_t * protocoldata = NULL;
    int fn = struct_field(stream, sz);
     ...
     ...
    return s;
}
```

在 create_from_bundle中 首先调用了struct_field(const uint8_t * stream, size_t sz) 取了 stream的头两个字节， 就是消息类型的个数fn， 即上篇中的 “type”和 “protocol” 这里是2，然后把 stream = stream + SIZEOF_HEADER + SIZEOF_FIELD * fn(这里fn是2); 移动到了消息体开始地方。在例子中我们传入的pbin串是450，加完之后的stream 移动到了第6个字节开始的地方， 此时的sz 是444 ，下面看代码注释

```c++
static int
struct_field(const uint8_t * stream, size_t sz) {
    const uint8_t * field;
    int fn, header, i;
    if (sz < SIZEOF_LENGTH)
        return -1;
    fn = toword(stream); //取了 头两个字节 此时fn =2
    header = SIZEOF_HEADER + SIZEOF_FIELD * fn; // SIZEOF_FIELD * fn; 
    //应该是协议内容"type"的长度 和"protocol" 的长度
    if (sz < header)
        return -1;
    field = stream + SIZEOF_HEADER; //field是扣除头部两个字节的
    sz -= header; //此时size = 444
    stream += header; // 此时的stream 删除了头部连个字节类型的个数信息 + 
    //SZIEOF_FIELD *fn 4个字节的的 "type"长度 和"protocol"长度
    for (i=0;i<fn;i++) {
        int value= toword(field + i * SIZEOF_FIELD);
        uint32_t dsz;
        if (value != 0)
            continue;
        if (sz < SIZEOF_LENGTH)
            return -1;
        // 如果value是0 那么在往后取2个字节  此时的stream 移动到了第8个字节处
        // 不懂为什么 value是会0.。..有大牛看到后求解释
        dsz = todword(stream);
        if (sz < SIZEOF_LENGTH + dsz)
            return -1;
        stream += SIZEOF_LENGTH + dsz;
        sz -= SIZEOF_LENGTH + dsz;
        //这里运算完， stream 移动到了 第二个类型的内容起开始， 即是
        //例子中 我们"protocol"内容的开始点
    }
    //for循环玩 stram 到了结尾 且sz = 0
    return fn; //返回了类型的个数。既1个 "type" 和 1个 "protocol"
}
```

回到create_from_bundle 中

```c++
create_from_bundle(struct sproto *s, const uint8_t * stream, size_t sz) {
    ...
    int fn = struct_field(stream, sz);
    stream += SIZEOF_HEADER;
    const uint8_t * content = stream + fn*SIZEOF_FIELD;  
    // content 位置就是struct_field 的field 此时 content 是 第6个字节
     ...

    // 接着进入了一个for循环 
    for (i=0;i<fn;i++) {
        int value = toword(stream + i*SIZEOF_FIELD); 
        //这里value 是第6，第7个字节，就是 "type"  内容的长度， 第二次循环是"protocol"长度
        int n;
        if (value != 0)
            return NULL;
        n = count_array(content); //计算出了"type" 内容有多少个元素， 第二此循环是 "protocol"元素的个数
        if (n<0)
            return NULL;
        if (i == 0) {
            typedata = content+SIZEOF_LENGTH; //typedata 扣除了 "type"长度的2个字节
            s->type_n = n;
            s->type = pool_alloc(&s->memory, n * sizeof(*s->type));
            // pool_alloc 为协议申请8倍数的内存
        } else {
            protocoldata = content+SIZEOF_LENGTH;//protocoldata 扣除了 "protocol"长度的2个字节
            s->protocol_n = n;
            s->proto = pool_alloc(&s->memory, n * sizeof(*s->proto));
        }
        content += todword(content) + SIZEOF_LENGTH; 
    }
    //content 循环结束就到结尾了
     ...
    return s;
}

```

让我们来看看 count_array 是怎么计算个数的

```
static int
count_array(const uint8_t * stream) {
    uint32_t length = todword(stream);
    int n = 0;
    stream += SIZEOF_LENGTH; //加了2个字节的长度信息，这地是第一层次加
    是整个"type"的长度信息
    while (length > 0) {
        uint32_t nsz;
        if (length < SIZEOF_LENGTH)
            return -1;
        nsz = todword(stream);//这里取的是第一个字节的长度
        nsz += SIZEOF_LENGTH; // 加2个字节
        if (nsz > length)
            return -1;
        ++n;
        stream += nsz; //加内容长度
        length -= nsz;
    }

    return n;

```

看完 count_array 我们明白了， 原来bin串就是序列化了的 **长度** 加 **内容** 的串create_from_bundle 中第一个for 循环结束后，typedata内容的其实地址设置好了。 protocoldata内容的起始地址也知道了 元素个数也知道了 内存也分配好了 之后就是两个for循环进行初始化了

```c++
static int
count_array(const uint8_t * stream) {
    . . .
        content += todword(content) + SIZEOF_LENGTH;
    }

    for (i=0;i<s->type_n;i++) {
        typedata = import_type(s, &s->type[i], typedata);
        //import_type 初始化 "type" 内容 
        if (typedata == NULL) {
            return NULL;
        }
    }
    for (i=0;i<s->protocol_n;i++) {
        protocoldata = import_protocol(s, &s->proto[i], protocoldata);
        //import_protocol初始化 "protocol" 内容
        if (protocoldata == NULL) {
            return NULL;
        }
    }

    return s;
}
```

在 import_type 和 import_protocol 之后，就把所有的协议信息都放到 sproto结构里面了。

bin = 2字节类型个数 + 4留空字节 + 2字节”type”内容长度 + （”type”内容 typedata）+ 2字节”protocol”长度 + （”protocol”内容 protocoldata）

typedata = { { 2字节元素总长度 + 2字节子元素个数信息 +4字节的留空字节 + 2个字节的名字长度信息 + 名字长度 + {子元素内容}}, {“同第一个”}， … }

例如 
.AddressBook { 
person 0 : *Person(id) 
others 1 : *Person 
}

名字长度就是 AddressBook 长度 ， 子元素内容就是 里面的 person 和 others

{子子元素内容} = 2字节的长度 + 内容