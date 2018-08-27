---
title: lua-1
date: 2018-08-27 09:11:06
tags: lua
categories: lua
---

# 简介

Lua 是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。。Lua是类C的，所以，他是大小写字符敏感的。

# Lua 环境安装

## Linux 系统上安装

Linux & Mac上安装 Lua 安装非常简单，只需要下载源码包并在终端解压编译即可。

```shell
wget https://www.lua.org/ftp/lua-5.3.5.tar.gz
tar zxf lua-5.3.5.tar.gz
cd lua-5.3.5/
make linux test
sudo make install
```

# 编写Lua脚本的helloworld

 Lua脚本的语句的分号是可选的，这个和GO语言很类似。

- 在hello.lua中编写lua代码

```lua
vim hello.lua
print("Hello World")
lua hello.lua
```

- 我们也可以将代码修改为如下形式来执行脚本（在开头添加：#!/usr/local/bin/lua）：

```lua
#!/usr/local/bin/lua

print("Hello World！")
print("www.runoob.com")
```

```shell
chmod 766 hell.lua
./hello.lua
```

- 你可以像python一样，在命令行上运行lua命令后进入lua的shell中执行语句。

Lua 交互式编程模式可以通过命令 lua -i 或 lua 来启用：

> ~ lua
> Lua 5.3.5  Copyright (C) 1994-2018 Lua.org, PUC-Rio
>
> \>print("hello world")
>
> hello world



# Lua 基本语法

## 注释

### 单行注释

两个减号是单行注释:

```
--
```

### 多行注释

```
--[[
 多行注释
 多行注释
 --]]
```

------

## 标示符

Lua 标示符用于定义一个变量，函数获取其他用户定义的项。标示符以一个字母 A 到 Z 或 a 到 z 或下划线 _ 开头后加上0个或多个字母，下划线，数字（0到9）。

## 关键词

以下列出了 Lua 的保留关键字。保留关键字不能作为常量或变量或其他用户自定义标示符：

| and      | break | do    | else   |
| -------- | ----- | ----- | ------ |
| elseif   | end   | false | for    |
| function | if    | in    | local  |
| nil      | not   | or    | repeat |
| return   | then  | true  | until  |
| while    |       |       |        |

一般约定，以下划线开头连接一串大写字母的名字（比如 _VERSION）被保留用于 Lua 内部全局变量。



# Lua 变量

**Lua 变量有三种类型：全局变量、局部变量、表中的域。**

## 全局变量

需要注意的是：lua中的变量如果没有特殊说明，全是全局变量，那怕是语句块或是函数里。变量前加**local**关键字的是局部变量。

全局变量不需要声明，给一个变量赋值后即创建了这个全局变量，访问一个没有初始化的全局变量也不会出错，只不过得到的结果是：nil。

```lua
> print(b)
nil
> b=10
> print(b)
10
> 
```

如果你想删除一个全局变量，只需要将变量赋值为nil。

```lua
b = nil
print(b)      --> nil
```

这样变量b就好像从没被使用过一样。换句话说, 当且仅当一个变量不等于nil时，这个变量即存在。

## 控制语句

##### while循环

```lua
sum = 0
num = 1
while num <= 100 do
    sum = sum + num
    num = num + 1
end
print("sum =",sum)

```

##### if-else分支

```lua
if age == 40 and sex =="Male" then
    print("男人四十一枝花")
elseif age > 60 and sex ~="Female" then
    print("old man without country!")
elseif age < 20 then
    io.write("too young, too naive!\n")
else
    local age = io.read()
    print("Your age is "..age)
end

```

上面的语句不但展示了if-else语句，也展示了
1）“～=”是不等于，而不是!=
2）io库的分别从stdin和stdout读写的read和write函数
3）字符串的拼接操作符“..”

 **（注意：Lua没有++或是+=这样的操作）**

条件表达式中的与或非为分是：and, or, not关键字。

##### for 循环

从1加到100

```lua
sum = 0
for i = 1, 100 do
    sum = sum + i
end
```

从1到100的奇数和

```lua
sum = 0
for i=1,100,2 do
	sum = sum +i
end
```

从100到1的偶数和

```lua
sum = 0
for i = 100, 1, -2 do
    sum = sum + i
end
```

##### until循环

 ```lua
sum = 2
repeat
   sum = sum ^ 2 --幂操作
   print(sum)
until sum >1000

 ```

## 函数

Lua的函数和Javascript的很像

### 递归

```lua
function fib(n)
  if n < 2 then return 1 end
  return fib(n - 2) + fib(n - 1)
end
```

### 闭包

```lua
function newCounter()
    local i = 0
    return function()     -- anonymous function
       i = i + 1
        return i
    end
end
 
c1 = newCounter()
print(c1())  --> 1
print(c1())  --> 2
```



















