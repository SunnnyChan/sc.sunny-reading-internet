### 《Lua 程序设计》[Reading Note] - 2 -函数
```md
By SunnyChan (sunnnychan@gmail.com)
2015-09-28
```

笔记没有完全按照书上的章节顺序进行记录，根据自己的学习目标做了一些调整。
Lua中函数的概念变化比较大，具有一些特有的处理机制，感觉是比较复杂的部分，所以单独做一些记录。

- Lua的函数实例：
```lua
function add(n)
    local sum = 0
    for i,v in ipairs(n) do
        sum = sum +v
    end
    return sum
end
```

- 函数调用：
```lua
add({1,3,4})
```

Lua的函数调用有一种特殊的情况：
一个函数若只有一个参数，并且实参为字面字符串或table构造式，则圆括号可以省略。
即上面的函数调用表达式也可以写成：add{1,3,4}

Lua中当函数调用时，如果实参和形成的数量不一致，此时用实参初始化形参的规则，与多重赋值是类似的：
“实参多余形参，多余部分舍弃；实参不足，多余的形参初始化为nil。”

- 多重返回值（multiple results）
Lua允许函数返回多个结果，如：
```lua
str_start, str_end = string.find("hello lua users", "lua")
```
string.find函数在字符串中查找指定的模式，返回匹配的起始字符和结束字符的索引。
```lua
lua>print(str_start, str_end)
7	9
```

Lua会调整函数的返回数量以适应不同的调用情况：
若将函数调用作为一条单独语句，则丢弃所有的返回值；
若将函数作为表达式的一部分来调用时，Lua只保留第一个返回值；
若函数调用时最后的（或仅有的）一个表达式，那么Lua会保留其尽可能多的值，用于匹配变量。
    如果函数没有返回足够多的值来赋值变量，多余的变量用nil来赋值。
    如果函数调用不是一系列表达式的最后一个元素，那么将只参数一个值：
```lua
    x, y = foo(), 20 -- x等于foo函数第一个返回值，y 等于 20
```

当一个函数调用作为另一个函数调用的最后一个实参时，那么它的所有返回值都将作为实参传入第二个函数。
```lua
lua>print(string.find('hello lua users','lua'))
7	9
```
如果，第二个函数是固定数量参数，Lua将第一个函数的返回值调整为第二个函数的参数个数。

table构造式可以完成的接收一个函数调用的所有结果。如：
```lua
lua>t = {string.find('hello lua users','lua')}
lua>print(t[1])
7
lua>print(t[2])
9
```
但是，这种情况只有当函数调用作为最后一个元素时才会发生，其他位置上的函数调用总是只产生一个结果值。

另外，如果将函数调用放入一对圆括号中，将强制返回一个结果：
```lua
lua>print( (string.find('hello lua users','lua')) )
7
```
这里要注意，return语句后面的内容是不需要圆括号的，如果加上圆括号可能导致不同的行为。如：
```lua
return (f(x))，将只返回一个值。
```

- 变长参数（variable number of arguments）
```lua
function accumulate(...)
    local sum = 0 
    for i, v in ipairs({...}) do
        sum = sum + v 
    end
    return sum
end

lua>print(accumulate(1, 3, 4))
8
```

Lua 中的变长参数用“...”来表示，访问变长参数时，3个点号是作为一个表达式来使用的，该表达式的行为类似于一个具有多重返回值的函数。
可以用变长参数机制来模拟普通的参数传递：
```lua
function foo(a, b,c) 可以转换为：
function foo(...)
    local a, b, c = ...
end
```

具有变长参数的函数也可以拥有任意数量的固定参数，但必须放在比变长参数前面。

如果变长参数中含有nil，就需要使用select来遍历：
```lua
select(selector, ...)
```
select函数通过指定selector（选择器）来访问变长参数，
```lua
for i = 1, select("#", ...) do
    local arg = select(i, ...)
end
```
如果selector为数字n，select返回第n个可变实参；
如果使字符串“#”，返回变长参数的总数，包括其中的nil。

- 具名实参（named arguments）
目前大多数语言的参数传递机制是通过形参和实参的位置来匹配传参的，有时候参数多了，就会产生混乱。
Lua可以通过table结构来传参，实现具名实参，使代码更具可读性，也避免在编码过程中，由于参数太多造成的混乱。
例如实现重命名文件：
```lua
function rename(arg)
    return os.rename(arg.old, arg.new)
end
调用：rename{old = "temp.lua", new ="temp1.lua" } --参数只有一个table构造式，函数调用中的圆括号可以省略。
```

- Lua函数中的新特性
Lua中，函数是一种“第一类值（First-Class Value）”，即函数与其他传统类型的值（如数字和字符串）具有相同的权利。
可以存储到变量中或table中，可以作为实参，还可以作为其他函数的返回值。
注意：Lua中函数与其他值一样都是匿名的，即它们都没有名称。当讨论一个函数名时，实际上是在讨论一个持有某函数的变量。
```lua
lua>echo =  print
lua>echo(1+1)
2
lua>print=nil
lua>print(1+1)
stdin:1: attempt to call global 'print' (a nil value)
stack traceback:
stdin:1: in main chunk
[C]: ?
```

实际上， function foo(x) return 2*x end 只是一种所谓的“语法糖”而已。只是以下代码的一种简化书写形式：
```lua
foo = function (x) return 2*x end
```
因此一个函数定义就是一条赋值语句，这条语句创建了一种类型为”函数“的值，并将这个值赋予一个变量。
可以将  function (x) return 2*x end视为函数的构造式，类似于table。将这个函数构造式的记过称为一个“匿名函数”。

>“高阶函数”

如果一个函数接受另一个函数作为实参，则称此函数为“高阶函数（higher-order function）”
高阶函数是一种强大的编程机制，应用匿名函数来创建高阶函数所需的实参可以带来很大的灵活性。但是高阶函数并没有任何的特权。

> 闭合函数（closure）

Lua函数具有特定的“词法域（Lexical Scoping）”，即一个函数可以嵌套在另一个函数中，内部的函数可以方位外部函数中的变量。
```lua
function counter()
    local i = 0 
    return function () 
        i = i + 1 
        return i 
        end
end

调用：
lua>func = counter()
lua>print(func())
1
lua>print(func())
2
lua>func_new = counter()
lua>print(func_new())
1
```
函数counter中的匿名函数下的i变量，既不是全局变量，也不是局部变量，称为“非局部变量（non-local variable）”。

i变量用于保持一个计数器，初看上去，由于创建i的函数已经返回，所有之后的每次调用匿名函数，i都应该超出了作用范围。

实际上，Lua以closure的概念来正确处理这种情况，简单理解一个closure就是一个函数加上该函数所需访问的所有“非局部变量”。

如果再次调用counter，会创建一个新的局部变量i，也将得到一个新的closure。

从技术上讲，Lua中只有closure，而不存在“函数”。因为，函数本身就是一个特殊的closure。
closure有很多应用场景，包括作为高阶函数的参数，作为其他函数的一部分来创建新函数，回调函数，以及重定义某些函数，甚至是预定义的函数。

> 非全局函数（non-global function）

将一个函数存储到一个局部变量，就可以得到一个局部函数“local function”，该函数只在某个特定的作用域才能调用。
前面碰到的string.find就是把函数存储在table的字段中，大部分的Lua库都采用这种机制。

局部函数的定义方式：
```lua
local f =  function () end
local function f() end
```

在定义地柜的局部函数时，需注意：
```lua
loca fact = function(n)
    if n == 0 then return 1
    else return n * fact(n-1) 
    end
end
```
以上代码，当执行fact(n-1)时，由于局部的fact尚未定义，因此调用的是全局的fact，而非函数自身。可以先定义局部变量：
```lua
local fact
fact = function(n)
    if n == 0 then return 1
    else return n * fact(n-1) 
    end
end
```

以上代码，fact 调用就表示了局部变量。即使在函数定义时，局部变量值未完成定义，但之后函数在执行时，fact则会拥有正确的值。
```lua
实际上，在Lua处理 local function f() end 时，会展开成 local f  f() end。
```
所以使用 local function fact(n)这样的定义方式是不会出错的。
但要注意：对于间接递归，这个技巧是无效的。在间接递归的情况中，必须使用一个明确的前向声明
（Forward Declaration）。
```lua
local f,  g
function g()
    ...   
    f()
    ...
end

function f()
    ...
    g()
    ...
end
```
- 正确的尾调用（proper tail call）

Lua支持“尾调用调出（tail-call elimination）”。它是类似于goto的函数调用。
当一个函数调用时另一个函数的最后一个动作时，才算是一条“尾调用”。如：
```lua
function f(x) return g(x) end
```
即当f调用完g之后就再无其他事情可做了，此时程序就不需要返回，所以在尾调用之后，程序不要保存任何关于该函数的栈信息。
当函数g返回时，可以直接返回到调用f的那个点上。
这样的语言实现，使得在进行“尾调用”时不会耗费任何栈空间，将这样的实现称为支持“尾调用消除”。

基于以上的特点，Lua中一个程序可以拥有无数嵌套的“尾调用”，是不会造成栈溢出的。如，调用以下函数，输入任何数字都不会出问题。
```lua
function foo()
    if n > 0 then return foo(n-1) end
end
```

注意：

在Lua中，只有 “return <func>(<args>)”这样的调用形式才算是一条“尾调用”。

Lua会再调用前对func及其参数求值，所以它可以是任意复杂的表达式，
如下面这条语句就是一个"尾调用"：
```lua
return x[i].foo(x[j] + a*b, i + j)
```

Lua中“尾调用”的一大应用就是编写“状态机（state machine）”。
这类程序通常以一个函数来表示一个状态，改变状态就是goto到另一个特定的函数。