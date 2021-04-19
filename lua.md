### int lua_rawgetp (lua_State *L, int index, const void *p);
将t\[k\]压栈，不触发元方法

### void lua_pop(lua_State *L, int n)
从栈中弹出n个元素

### int luaL_newmetatable (lua_State *L, const char *tname);
如果注册表中已存在键tname，则返回0；否则为用户数据的原表创建一张新表、向这张表加入__name=tname键值对，并将\[tname\]=new table添加到注册表中，返回1.（__name项可用于一些错误输出函数）

### void luaL_setmetatable (lua_State *L, const char *tname); 
将注册表中tname关联栈顶对象的元表

### void *luaL_testudata (lua_State *L, int arg, const char *tname);
此函数和luaL_checkudata类似、但在失败时会返回NULL而不是抛出错误

### void *luaL_checkudata (lua_State *L, int arg, const char *tname);
检查函数的第arg个参数是否是一个类型为tname的用户数据。它会返回该用户数据的地址。(参见lua_touserdata)

### void *lua_touserdata (lua_State *L, int index);
如果给定索引处的值是一个完全用户数据，函数就返回其内存块的地址。如果是一个轻量级用户数据，那么就返回它代表的指针，否则返回NULL

### void lua_pushvalue (lua_State *L, int index);
将栈上索引处的元素做一个副本压栈

### void luaL_checkstack (lua_State *L, int sz, const char *msg);
将栈空间扩展到top+sz个元素。如果扩展不了就跑出一个错误。

### int lua_next (lua_State *L, int index);
从栈顶弹出一个键，然后把索引指定的表中的一个键值对压栈（弹出键之后的“下一”对）。如果表中无更多元素，那么返回0（什么也不压栈）
典型的遍历方法如下:
```
// 表放在索引't'处
lua_pushnil(L); // 第一个键
while(lua_next(L, t) != 0) {
    // 使用键(-2)、值(-1)
    printf("%s - %s\n", lua_typename(L, lua_type(L, -2), lua_typename(L, lua_type(L, -1))));
    lua_pop(L, 1);
}
```
在遍历一张表的时候，不要直接对键调用lua_tolstring，除非确认这个值是一个string、否则可能改变索引位置的值，对下一次调用lua_next产生影响

### const char *lua_tolstring (lua_State *L, int index, size_t *len);
    把给定索引处的lua值转换为一个c字符串、如果len部位NULL，还将长度设到*len中。这个lua值必须是一个字符串或者数字

### setjmp和logjmp是配合使用的，用它们可以实现跳转的功能，和goto语句很类似，不同的是goto只能实现在同一个函数之内的跳转，而setjmp和logjmp可以实现在不同函数间的跳转