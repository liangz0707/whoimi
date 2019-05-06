# Lua 语言 15 分钟快速入门

本文来自[oschina.net ](http://www.linuxidc.com/Linux/2013-06/86582.htm)

[back](../../../index.md)

[back to language](../lang_index.md)

### 注释

```lua
-- 两个横线开始单行的注释
--[[
    加上两个[和]表示
      多行的注释。
 --]]
```

### 变量 

```lua
num = 42  -- 所有的数字都是double。 
-- 别担心，double的64位中有52位用于 
-- 保存精确的int值; 对于需要52位以内的int值， 
-- 机器的精度不是问题。

s = 'walternate'  -- 像Python那样的不可变的字符串。 
t = "双引号也可以" 
u = [[ 两个方括号 
      用于 
      多行的字符串。]] 
t = nil  -- 未定义的t; Lua 支持垃圾收集。
```

### 流控制

```lua
-- do/end之类的关键字标示出程序块： 
while num < 50 do 
  num = num + 1  -- 没有 ++ or += 运算符。 
end

-- If语句： 
if num > 40 then 
  print('over 40') 
elseif s ~= 'walternate' then  -- ~= 表示不等于。 
  -- 像Python一样，== 表示等于；适用于字符串。 
  io.write('not over 40\n')  -- 默认输出到stdout。 
else 
  -- 默认变量都是全局的。 
  thisIsGlobal = 5  -- 通常用驼峰式定义变量名。

  -- 如何定义局部变量： 
  local line = io.read()  -- 读取stdin的下一行。

  -- ..操作符用于连接字符串： 
  print('Winter is coming, ' .. line) 
end

-- 未定义的变量返回nil。 
-- 这不会出错： 
foo = anUnknownVariable  -- 现在 foo = nil.

aBoolValue = false

--只有nil和false是fals; 0和 ''都是true！ 
if not aBoolValue then print('twas false') end

-- 'or'和 'and'都是可短路的（译者注：如果已足够进行条件判断则不计算后面的条件表达式）。 
-- 类似于C/js里的 a?b:c 操作符： 
ans = aBoolValue and 'yes' or 'no'  --> 'no'

karlSum = 0 
for i = 1, 100 do  -- 范围包括两端 
  karlSum = karlSum + i 
end

-- 使用 "100, 1, -1" 表示递减的范围： 
fredSum = 0 
for j = 100, 1, -1 do fredSum = fredSum + j end

-- 通常，范围表达式为begin, end[, step].

-- 另一种循环表达方式： 
repeat 
  print('the way of the future') 
  num = num - 1 
until num == 0
```

###  函数

```lua
function fib(n) 
  if n < 2 then return 1 end 
  return fib(n - 2) + fib(n - 1) 
end

-- 支持闭包及匿名函数： 
function adder(x) 
  -- 调用adder时，会创建用于返回的函数，并且能记住变量x的值： 
  return function (y) return x + y end 
end 
a1 = adder(9) 
a2 = adder(36) 
print(a1(16))  --> 25 
print(a2(64))  --> 100

-- 返回值、函数调用和赋值都可以使用长度不匹配的list。 
-- 不匹配的接收方会被赋为nil； 
-- 不匹配的发送方会被忽略。

x, y, z = 1, 2, 3, 4 
-- 现在x = 1, y = 2, z = 3, 而 4 会被丢弃。

function bar(a, b, c) 
  print(a, b, c) 
  return 4, 8, 15, 16, 23, 42 
end

x, y = bar('zaphod')  --> prints "zaphod  nil nil" 
-- 现在 x = 4, y = 8, 而值15..42被丢弃。

-- 函数是一等公民，可以是局部或者全局的。 
-- 下面是等价的： 
function f(x) return x * x end 
f = function (x) return x * x end

-- 这些也是等价的： 
local function g(x) return math.sin(x) end 
local g; g  = function (x) return math.sin(x) end 
-- 'local g'可以支持g自引用。

-- 顺便提一下，三角函数是以弧度为单位的。

-- 用一个字符串参数调用函数，不需要括号： 
print 'hello'  --可以工作。
```

### Table

```lua
-- Table = Lua唯一的数据结构; 
--        它们是关联数组。 
-- 类似于PHP的数组或者js的对象， 
-- 它们是哈希查找表（dict），也可以按list去使用。

-- 按字典/map的方式使用Table：

-- Dict的迭代默认使用string类型的key： 
t = {key1 = 'value1', key2 = false}

-- String的key可以像js那样用点去引用： 
print(t.key1)  -- 打印 'value1'. 
t.newKey = {}  -- 添加新的 key/value 对。 
t.key2 = nil  -- 从table删除 key2。

-- 使用任何非nil的值作为key： 
u = {['@!#'] = 'qbert', [{}] = 1729, [6.28] = 'tau'} 
print(u[6.28])  -- 打印 "tau"

-- 对于数字和字符串的key是按照值来匹配的，但是对于table则是按照id来匹配。 
a = u['@!#']  -- 现在 a = 'qbert'. 
b = u[{}]    -- 我们期待的是 1729,  但是得到的是nil: 
-- b = nil ，因为没有找到。 
-- 之所以没找到，是因为我们用的key与保存数据时用的不是同一个对象。 
-- 所以字符串和数字是可用性更好的key。

-- 只需要一个table参数的函数调用不需要括号： 
function h(x) print(x.key1) end 
h{key1 = 'Sonmi~451'}  -- 打印'Sonmi~451'.

for key, val in pairs(u) do  -- Table 的遍历. 
  print(key, val) 
end

-- _G 是一个特殊的table，用于保存所有的全局变量 
print(_G['_G'] == _G)  -- 打印'true'.

-- 按list/array的方式使用：

-- List 的迭代方式隐含会添加int的key： 
v = {'value1', 'value2', 1.21, 'gigawatts'} 
for i = 1, #v do  -- #v 是list的size 
  print(v[i])  -- 索引从 1 开始!! 太疯狂了！ 
end 
-- 'list'并非真正的类型，v 还是一个table， 
-- 只不过它有连续的整数作为key，可以像list那样去使用。
```

### 元表(metatable)

```lua
-- table的元表提供了一种机制，可以重定义table的一些操作。 
-- 之后我们会看到元表是如何支持类似js的prototype行为。

f1 = {a = 1, b = 2}  -- 表示一个分数 a/b. 
f2 = {a = 2, b = 3}

-- 这个是错误的： 
-- s = f1 + f2

metafraction = {} 
function metafraction.__add(f1, f2) 
  sum = {} 
  sum.b = f1.b * f2.b 
  sum.a = f1.a * f2.b + f2.a * f1.b 
  return sum 
end

setmetatable(f1, metafraction) 
setmetatable(f2, metafraction)

s = f1 + f2  -- 调用在f1的元表上的__add(f1, f2) 方法

-- f1, f2 没有能访问它们元表的key，这与prototype不一样， 
-- 所以你必须用getmetatable(f1)去获得元表。元表是一个普通的table， 
-- Lua可以通过通常的方式去访问它的key，例如__add。

-- 不过下面的代码是错误的，因为s没有元表： 
-- t = s + s 
-- 下面的类形式的模式可以解决这个问题：

-- 元表的__index 可以重载点运算符的查找： 
defaultFavs = {animal = 'gru', food = 'donuts'} 
myFavs = {food = 'pizza'} 
setmetatable(myFavs, {__index = defaultFavs}) 
eatenBy = myFavs.animal  -- 可以工作！这要感谢元表的支持

-- 如果在table中直接查找key失败，会使用元表的__index 继续查找，并且是递归的查找

-- __index的值也可以是函数function(tbl, key) ，这样可以支持更多的自定义的查找。

-- __index、__add等等，被称为元方法。 
-- 这里是table的元方法的全部清单：

-- __add(a, b)                    for a + b 
-- __sub(a, b)                    for a - b 
-- __mul(a, b)                    for a * b 
-- __div(a, b)                    for a / b 
-- __mod(a, b)                    for a % b 
-- __pow(a, b)                    for a ^ b 
-- __unm(a)                        for -a 
-- __concat(a, b)                  for a .. b 
-- __len(a)                        for #a 
-- __eq(a, b)                      for a == b 
-- __lt(a, b)                      for a < b 
-- __le(a, b)                      for a <= b 
-- __index(a, b)  <fn or a table>  for a.b 
-- __newindex(a, b, c)            for a.b = c 
-- __call(a, ...)                  for a(...)
```



### 类风格的table

```lua
-- 类并不是内置的；有不同的方法通过表和元表来实现。

-- 下面是一个例子，后面是对例子的解释

Dog = {}                                  -- 1.

function Dog:new()                        -- 2. 
  newObj = {sound = 'woof'}                -- 3. 
  self.__index = self                      -- 4. 
  return setmetatable(newObj, self)        -- 5. 
end

function Dog:makeSound()                  -- 6. 
  print('I say ' .. self.sound) 
end

mrDog = Dog:new()                          -- 7. 
mrDog:makeSound()  -- 'I say woof'        -- 8.

-- 1. Dog看上去像一个类；其实它完全是一个table。 
-- 2. 函数tablename:fn(...) 与函数tablename.fn(self, ...) 是一样的 
--    冒号（:）只是添加了self作为第一个参数。 
--    下面的第7和第8条说明了self变量是如何得到其值的。 
-- 3. newObj是类Dog的一个实例。 
-- 4. self为初始化的类实例。通常self = Dog，不过继承关系可以改变这个。 
--    如果把newObj的元表和__index都设为self， 
--    newObj就可以得到self的函数。 
-- 5. 记住：setmetatable返回其第一个参数。 
-- 6. 冒号（：）在第2条是工作的，不过这里我们期望 
--    self是一个实例，而不是类 
-- 7. 与Dog.new(Dog)类似，所以 self = Dog in new()。 
-- 8. 与mrDog.makeSound(mrDog)一样; self = mrDog。

```

### 继承

```lua
LoudDog = Dog:new()                          -- 1.

function LoudDog:makeSound() 
  s = self.sound .. ' '                      -- 2. 
  print(s .. s .. s) 
end

seymour = LoudDog:new()                      -- 3. 
seymour:makeSound()  -- 'woof woof woof'      -- 4.

-- 1. LoudDog获得Dog的方法和变量列表。 
-- 2. 通过new()，self有一个'sound'的key from new()，参见第3条。 
-- 3. 与LoudDog.new(LoudDog)一样，并且被转换成 
--    Dog.new(LoudDog)，因为LoudDog没有'new' 的key， 
--    不过在它的元表可以看到 __index = Dog。 
--    结果: seymour的元表是LoudDog，并且 
--    LoudDog.__index = LoudDog。所以有seymour.key 
--    = seymour.key, LoudDog.key, Dog.key, 要看 
--    针对给定的key哪一个table排在前面。 
-- 4. 在LoudDog可以找到'makeSound'的key；这与 
--    LoudDog.makeSound(seymour)一样。

-- 如果需要，子类也可以有new()，与基类的类似： 
function LoudDog:new() 
  newObj = {} 
  -- 初始化newObj 
  self.__index = self 
  return setmetatable(newObj, self) 
end
```

### 模块

```lua
--[[ 我把这部分给注释了，这样脚本剩下的部分就可以运行了

-- 假设文件mod.lua的内容是： 
local M = {}

local function sayMyName() 
  print('Hrunkner') 
end

function M.sayHello() 
  print('Why hello there') 
  sayMyName() 
end

return M

-- 另一个文件也可以使用mod.lua的函数： 
local mod = require('mod')  -- 运行文件mod.lua.

-- require是包含模块的标准做法。 
-- require等价于:    (针对没有被缓存的情况；参加后面的内容) 
local mod = (function () 
  <contents of mod.lua> 
end)() 
-- mod.lua就好像一个函数体，所以mod.lua的局部变量对外是不可见的。

-- 下面的代码是工作的，因为在mod.lua中mod = M： 
mod.sayHello()  -- Says hello to Hrunkner.

-- 这是错误的；sayMyName只在mod.lua中存在： 
mod.sayMyName()  -- 错误

-- require返回的值会被缓存，所以一个文件只会被运行一次， 
-- 即使它被require了多次。

-- 假设mod2.lua包含代码"print('Hi!')"。 
local a = require('mod2')  -- 打印Hi! 
local b = require('mod2')  -- 不再打印; a=b.

-- dofile与require类似，只是不做缓存： 
dofile('mod2')  --> Hi! 
dofile('mod2')  --> Hi! (再次运行，与require不同)

-- loadfile加载一个lua文件，但是并不允许它。 
f = loadfile('mod2')  -- Calling f() runs mod2.lua.

-- loadstring是loadfile的字符串版本。 
g = loadstring('print(343)')  --返回一个函数。 
g()  -- 打印343; 在此之前什么也不打印。

--]]
```

