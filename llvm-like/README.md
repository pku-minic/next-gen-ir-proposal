# 类 LLVM 的 IR

我们暂且把这种 IR 叫做 IR2.

## 符号名称

### 说明

符号名称可用来表示变量, 函数或标号. 符号分为以下两种:

* **具名符号**: `@name`, `name` 可以是任意可被 `[_A-Za-z][_A-Za-z0-9]*` 匹配的字符串, 也就是类 C 语言的标识符.
* **临时符号**: `%name`, `name` 可以是任意可被 `[0-9]|[1-9][0-9]+|[_A-Za-z][_A-Za-z0-9]*` 匹配的字符串, 也就是类 C 语言的标识符或者任意数字.

这两种符号没有任何区别, 只是我们通常用前一种表示原语言中的变量或函数, 而用后一种类型来表示临时值或标号.

让临时符号也可以包括字母和下划线的目的是, 方便大家调试. 这样你就不需要对着一堆数字来判断到底哪一个标号是 `while` 循环的结束了.

## 类型

### 语法

```ebnf
Shape ::= "[" INT "]" [Shape];
```

### 说明

IR2 的表达式或值可返回以下类型:

* **32 位有符号整数**: 默认类型.
* **数组**: 可用 `Shape` 来描述数组的 shape, 如:
  * `[2]`: 长度为 2 的一维 32 位有符号整数数组.
  * `[2][3]`: 长度分别为 2 和 3 的二维 32 位有符号整数数组.

简单起见, IR2 在书写时不需要标注类型声明, 程序员需要确保 IR2 在形式上不具备任何类型错误, 否则 IR2 程序的行为是未定义的.

当然, IR2 在解析的时候, 解析器应当试图找出 IR2 程序中潜在的类型问题, 并报告.

> **TODO:** 不标注类型声明的行为只在现阶段的课程中适用, 因为现阶段课程所需要实现的原语言并不打算支持除 32 位有符号整数和数组外的其他类型.
>
> 为了可持续发展, 我们是否应该牺牲一部分 IR 的简洁性?

## 值

### 语法

```ebnf
Value ::= SYMBOL | INT;
Initializer ::= INT | Aggregate | "zeroinit";
Aggregate ::= Shape "{" Initializer {"," Initializer} "}";
```

### 说明

* **32 位有符号整数**: 形如 `1`, `233` 等.
* **符号引用**: 形如 `@var`, `%0` 等.
* **数组初始化列表**: 形如 `[3] {1, 2, 3}`, `[2][2] {{1, 2}, {3, 4}}` 等, 只允许用来初始化数组类型.
* **零初始化器**: `zeroinit`, 可以初始化任何类型.

## 符号定义

### 语法

```ebnf
SymbolDef ::= SYMBOL "=" (MemoryDeclaration | Load | GetPointer | BinaryExpr | UnaryExpr | FunCall | GetArg);
GlobalSymbolDef ::= "global" SYMBOL "=" GlobalMemoryDeclaration;
```

### 说明

`SymbolDef` 只能出现在函数内部, `GlobalSymbolDef` 只能出现在函数外部.

所有的符号都应该只被定义一次.

## 内存声明

### 语法

```ebnf
MemoryDeclaration ::= "alloc" [Shape];
GlobalMemoryDeclaration ::= "alloc" [Shape] "," Initializer;
```

### 示例

```ir2
@i = alloc                                // int i
@arr = alloc [2][3]                       // int arr[2][3]
global @arr2 = alloc [2][5], zeroinit     // int arr2[2][5] = {}
global @arr3 = alloc [3], [3] {1, 2, 3}   // int arr3[3] = {1, 2, 3}
```

## 内存访问

### 语法

```ebnf
Load ::= "load" Value;
Store ::= "store" Initializer "," SYMBOL;
```

### 示例

```ir2
// x = i
%0 = load @i
store %0, @x
```

## 指针运算

### 语法

```ebnf
GetPointer ::= "getptr" SYMBOL "," Value ["," INT];
```

### 说明

第一个参数是表示原指针的符号, 第二个参数是 `index`, 第三个参数是步长.

默认情况下, 步长是 1, 表示 `index` 每增加 1, `getptr` 就会向后寻址 1 个 32 位有符号整数.

### 示例

普通的数组:

```ir2
// int a[10][9];
// a[2][3] = 5
%0 = getptr @a, 2, 9
%1 = getptr %0, 3
store 5, %1
```

作为参数, 省略第一维长度的数组:

```ir2
// int a[][9]
// a[2][3] = 5;
%0 = getptr @a, 2, 9
%1 = getptr %0, 3
store 5, %1
```

和前一个例子没有任何区别.

## 双目/单目运算

### 语法

```ebnf
BinaryExpr ::= BINARY_OP Value "," Value;
UnaryExpr ::= UNARY_OP Value;
```

### 说明

支持的操作:

* **双目**: `ne`, `eq`, `gt`, `lt`, `ge`, `le`, `land`, `lor`, `add`, `sub`, `mul`, `div`, `mod`.
* **单目**: `neg`, `lnot`.

### 示例

```ir2
%2 = add %0, %1
%3 = mul %0, %2
%4 = neg %3
```

## 分支和跳转

### 语法

```ebnf
Branch ::= "br" Value "," SYMBOL "," SYMBOL;
Jump ::= "jump" SYMBOL;
```

### 说明

`Branch` 的第一个参数是分支条件. 条件非 0 时, 将跳转到第二个参数代表的标号处, 否则跳转到第三个参数代表的标号处.

`Jump` 语句会直接将控制流转移到 `SYMBOL` 所代表的标号处.

### 示例

```ir2
%while_entry:
  %cond = lt %0, %1                   // while (%0 < %1)
  br %cond, %while_body, %while_end   // {

%while_body:
  ...                                 //   ...
  jump %while_entry

%while_end:                           // }
```

## 函数调用和返回

### 语法

```ebnf
FunCall ::= "call" SYMBOL "(" [Value {"," Value}] ")";
Return ::= "ret" [Value];
```

### 说明

`FunCall` 括号内的内容为函数调用时传递的参数.

### 示例

```ir2
%0 = call @getint ()
call @putint (%0)
ret %0
```

## 函数和参数

### 语法

```ebnf
FunDef ::= "fun" SYMBOL "(" INT ")" "{" FunBody "}";
FunBody ::= {Statement};
Statement ::= SymbolDef | Store | Branch | Jump | FunCall | Return;
GetArg ::= "getarg" INT;
```

> **TODO:** 是否要强制要求大家按照基本块的方式来组织代码?
>
> 这样, 每个基本块的结尾就必须是 `Branch`, `Jump` 或者 `Return`.
>
> 而且, `Branch` 的双目标设计也会变得有意义起来, 同时在 IR 上进行优化也会更加方便.
>
> IR 也可以兼容 SSA 形式.

### 说明

`FunDef` 用来定义一个函数, 其中的 `INT` 表示函数参数的个数.

`GetArg` 用来获取函数的某个参数.

### 示例

```ir2
global @arr = alloc [10], zeroinit    // int arr[10] = {};

fun @func (2) {                       // int func(int a, int b) {
%entry:
  @a = alloc
  @b = alloc
  %ret = alloc

  %0 = getarg 0
  store %0, @a
  %1 = getarg 1
  store %1, @a
  jump %2

%2:
  %3 = load @a                        //   return arr[a] + b;
  %4 = getptr @arr, %3
  %5 = load %4
  %6 = load @b
  %7 = add %5, %6
  store %7, %ret
  jump %end

%end:
  %8 = load %ret
  ret %8
}                                     // }
```

## TODO

> Phi, 注释和注解.

没写完, 有空再写, 慢慢来嘛, 不着急.

实验室蚊子太多, 不宜久留.

我先 TODO 为敬.
