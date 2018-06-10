## skynet sproto 协议的生成

**lsproto.c**		 //一些导出到lua使用的函数 

**sproto.c**		 //非导出的函数 

**sproto.lua** 	//封装导出的接口，lua中使用sproto require这个文件就好 

**sprotoparser.lua** 	//封装了 lpeg解析协议的接口 lpeg是用于文本匹配的表达式， sprotoparser通过lepg把原始的协议字符串, 解析成可被sproto识别的bin串 

**protoloader.lua**	 //这个文件在 skynet/lualib里面 负责多个luavm共享协议

假设你的协议如下

```lua
local type = [[
.Person {
    name 0 : string
    id 1 : integer
    email 2 : string

    .PhoneNumber {
        number 0 : string
        type 1 : integer
    }

    phone 3 : *PhoneNumber
}

.AddressBook {
    person 0 : *Person(id)
    others 1 : *Person
}

]]
local pro = [[
    testprot 1 {
        request {
            p       0 : Person
            addr    1 : AddressBook
        }
        response {
            ret      0 : integer
        }
    }
]]
```

首先协议用过sprotoparser解析成可被sproto识别的bin串

```lua
local parser= require "sprotoparser"
local sproto = require "sproto"
local pbin = parser.parse(type .. pro )
local proto = sproto.new(pbin)

--在sproto.lua里面也有parse函数 只是封装了一下sprotoparser.parse
function sproto.parse(ptext)
    local parser = require "sprotoparser"
    local pbin = parser.parse(ptext)
    return sproto.new(pbin)
end
--你也可以这样调用
local sproto = require "sproto"
sproto.parse(type .. pro )
```

parse 函数定义如下

```lua
function sparser.parse(text, name)
    local r = parser(text, name or "=text")
    local data = encodeall(r)
    return data
end

local function parser(text,filename)
    local state = { file = filename, pos = 0, line = 1 }
    local r = lpeg.match(proto * -1 + exception , text , 1, state )
    return flattypename(check_protocol(adjust(r)))
end
```

在parse里面首先调用 lpeg.match 把字符串解析成lua table 
比如我们上面的协议， match后就生成下边的table

```lua
table: 0x239b3e0 = {
    [1] = {
        [1] = "Person"
        [2] = {
            [1] = {
                ["type"] = "field"
                [1] = "name"
                [2] = 0
                [3] = "string"
            }
            [2] = {
                ["type"] = "field"
                [1] = "id"
                [2] = 1
                [3] = "integer"
            }
            [3] = {
                ["type"] = "field"
                [1] = "email"
                [2] = 2
                [3] = "string"
            }
            [4] = {
                [1] = "PhoneNumber"
                [2] = {
                    [1] = {
                        ["type"] = "field"
                        [1] = "number"
                        [2] = 0
                        [3] = "string"
                    }
                    [2] = {
                        ["type"] = "field"
                        [1] = "type"
                        [2] = 1
                        [3] = "integer"
                    }
                }
                ["type"] = "type"
            }
            [5] = {
                [1] = "phone"
                [2] = 3
                [3] = "*"
                [4] = "PhoneNumber"
                ["type"] = "field"
            }
        }
        ["type"] = "type"
    }
    [2] = {
        [1] = "AddressBook"
        [2] = {
            [1] = {
                [1] = "person"
                [2] = 0
                [3] = "*"
                [4] = "Person"
                ["type"] = "field"
                [5] = "id"
            }
            [2] = {
                [1] = "others"
                [2] = 1
                [3] = "*"
                [4] = "Person"
                ["type"] = "field"
            }
        }
        ["type"] = "type"
    }
    [3] = {
        ["type"] = "protocol"
        [1] = "testprot"
        [2] = 1
        [3] = {
            [1] = {
                [1] = "request"
                [2] = {
                    [1] = {
                        ["type"] = "field"
                        [1] = "p"
                        [2] = 0
                        [3] = "Person"
                    }
                    [2] = {
                        ["type"] = "field"
                        [1] = "addr"
                        [2] = 1
                        [3] = "AddressBook"
                    }
                }
            }
            [2] = {
                [1] = "response"
                [2] = {
                    [1] = {
                        ["type"] = "field"
                        [1] = "ret"
                        [2] = 0
                        [3] = "integer"
                    }
                }
            }
        }
    }
}
```

在上面的表中 带有 [“type”] = “type” 标签的是我们定义的结构体 

而带有[“type”] = “protocol” 的则是我们要通信发送的协议 

match 之后 在调用adjust 对生成表的表进行调整，

local result = { type = {} , protocol = {} } adjust把我们商标的表带type标签的都放到 result.type里面。 

而带protocol标签的 都放到了result.protocol里面。在放进去之前 调用了convert.type 和 convert.protocol 再次调整了上边打大表

convert.protocol 把带”protocol”标签的协议调整成

```lua
  ["protocol"] = {
        ["testprot"] = {
            ["request"] = "testprot.request"
            ["response"] = "testprot.response"
            ["tag"] = 1
        }
    }
```

并且放入了 result.protocol 而带 “type”标签的 调整成



["type"] = {
```lua
    ["AddressBook"] = {
        [1] = {
            ["typename"] = "Person"
            ["name"] = "person"
            ["array"] = true
            ["key"] = "id"
            ["tag"] = 0
        }
        [2] = {
            ["typename"] = "Person"
            ["array"] = true
            ["name"] = "others"
            ["tag"] = 1
        }
    }
    ["Person.PhoneNumber"] = {
        [1] = {
            ["typename"] = "string"
            ["name"] = "number"
            ["tag"] = 0
        }
        [2] = {
            ["typename"] = "integer"
            ["name"] = "type"
            ["tag"] = 1
        }
    }
    ["Person"] = {
        [1] = {
            ["typename"] = "string"
            ["name"] = "name"
            ["tag"] = 0
        }
        [2] = {
            ["typename"] = "integer"
            ["name"] = "id"
            ["tag"] = 1
        }
        [3] = {
            ["typename"] = "string"
            ["name"] = "email"
            ["tag"] = 2
        }
        [4] = {
            ["typename"] = "PhoneNumber"
            ["array"] = true
            ["name"] = "phone"
            ["tag"] = 3
        }
    }
    ["testprot.response"] = {
        [1] = {
            ["typename"] = "integer"
            ["name"] = "ret"
            ["tag"] = 0
        }
    }
    ["testprot.request"] = {
        [1] = {
            ["typename"] = "Person"
            ["name"] = "p"
            ["tag"] = 0
        }
        [2] = {
            ["typename"] = "AddressBook"
            ["name"] = "addr"
            ["tag"] = 1
        }
    }
}
```

并放入result.type中

调整之后， 在调用了 check_protocol（result） 检查协议是否调整成功， 在之后 调用了 flattypename 调整了嵌套的typename ，还是我们上边的大表， flattypename 之后 ，我们协议 .Person 的第四个元素的 typename 被补全为 [“typename”] =“Person.PhoneNumber” 原来是 [“typename”] = “PhoneNumber” 这样我就就知道 .PhoneNumber 这个结构体是在type 的.Person里面定义的机构， 发解析协议的时候就根据这个来解析。 而其他直接在type里面的结构， typename不变。 到这里 parse的工作就做完了，返回了一个区分 type 和 protocol的table  然后回到函数

```lua
function sparser.parse(text, name)
    local r = parser(text, name or "=text")
    local data = encodeall(r)
    return data
end
```

parser之后 把我们上面解析的 result 放入了 encodeall 进行序列化

```lua
local function encodeall(r)
    return packgroup(r.type, r.protocol)
end
```

encodeall 之后返回的就是 sproto.new(pbin) 需要的参数pbin了

```lua
O 
AddressBook6personothersnPersonZnameidemaiphoneNPerson.PhoneNumber.numbertypeGtestprot.request)paddr4testprot.responseret  
```

