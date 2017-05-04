---
title: redis-lua-script
date: 2017-05-04 20:07:39
tags: lua,redis,openresty
---

## redis的lua脚本拓展，返回nil及其判断
### OpenResty的lua_resty_redis返回为空的判断

```
local res, err = red:get("dog")
if not res then
    ngx.say("failed to get dog: ", err)
    return
end
if res == ngx.null then
    ngx.say('res is null')
else
    ngx.say('res is not empty')
end
ngx.say(type(res))
ngx.say(res)

```
结果：

```
res is null
userdata
null

```

+ lua_resty_redis的redis操作返回的数据打印出来是null，但是不等同于lua中的nil，也不能直接和null比较，里面是lua的8中数据类型中的userdata。文档中已经有描述了：A non-nil Redis "bulk reply" results in a Lua string as the return value. A nil bulk reply results in a ngx.null return value.  [lua_resty_redis文档](https://github.com/openresty/lua-resty-redis)

### redis自带的lua脚本

```
127.0.0.1:6379> hget team wyc
"{\"name\":\"wyycc\",\"age\":\"444\"}"
```
想要通过redis官方的lua脚本拓展来传递name,age等直接返回对应的值。如果传递name，age等redis的hash中存在的value，则返回其结果，如果不存在返回nil，使得传入和返回结果一一对应。

+ redis的lua拓展自带了cjson等库，可以很方便的处理json数据。
+ 当返回table中有nil时停止，后续的数据无法返回
+ nil在lua的table中相当删除某个key，table.insert()是无法插入到table中的
+ 想要返回nil，则插入lua的FALSE。Lua boolean false -> Redis Nil bulk reply

```
127.0.0.1:6379> eval "return {1,2,3.3333,'foo',nil,'bar'}" 0
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) "foo"

```
> 后面的0表明传入几个参数，KEYS，ARGV两个table中接受传入的参数

```
127.0.0.1:6379> eval "local substring = redis.call('hget',KEYS[1],KEYS[2]) if not substring then return false end substring = cjson.decode(substring) local result = {} for _, v in ipairs(ARGV) do table.insert(result,substring[v] or false) end return result" 2 team wyc age len name
1) "444"
2) (nil)
3) "wyycc"

```
