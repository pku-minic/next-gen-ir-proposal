# 基于栈的 IR, v0.1-d

我们暂且把这种 IR 叫做 S2.

S2 的设计参考了 [MiniVM](https://github.com/pku-minic/MiniVM) 内部所使用的 [Gopher](https://github.com/pku-minic/MiniVM/blob/master/src/vm/README.md).

## 指令格式

S2 IR 的指令由两个字段组成, 分别为**操作** (`op`) 和**操作数** (`opr`).

前者用来编码 IR 的操作, 后者用来编码这条指令所需要的其他必要的信息, 但未必是真的 “操作数”. 因为 S2 是一种基于栈的 IR, 大部分操作的操作数都来自于操作数栈的栈顶.

## 抽象机器

S2 IR 的语义和执行机制基于一种抽象机器, 该机器需要具备:

* **程序计数器**: 存储当前正在执行的指令的地址, 用于指导指令的获取.
* **操作数栈**: 存储 S2 IR 的操作数.
* **局部变量栈**: 存储函数的局部变量.
* **全局内存**: 存储全局变量.

在汇编代码生成阶段, 抽象机器会被映射到实际的指令系统体系结构上. 通常情况下:

* “程序计数器” 会被直接映射为目标指令系统的程序计数器.
* 全局内存” 会被映射为 `data` 段或者 `bss` 段所代表的内存.
* “操作数栈” 和 “局部变量栈” 会被映射在函数的栈帧中. 以 `riscv32` 架构为例, 映射后栈帧的结构如下:

| 地址      | 值                                            |
| --        | --                                            |
| sp        | 操作数栈的栈顶元素 (低地址)                   |
| sp+4      | 操作数栈的其余元素                            |
| ...       | ...                                           |
| fp-12-4n  | 第 n 个局部变量                               |
| ...       | ...                                           |
| fp-16     | 第 1 个局部变量                               |
| fp-12     | 第 0 个局部变量                               |
| fp-8      | 上一个函数的 fp (已保存的 callee save 寄存器) |
| fp-4      | 当前函数的返回地址                            |
| fp        | 上一个函数的栈帧                              |
| ...       | ... (高地址)                                  |

## 符号名称

同 [IR2 提案](/ir/llvm-like#符号名称). 符号名称只能被用来标记全局变量, 函数名称或标号.

## 局部/全局变量/数组声明

### 语法

```ebnf
LocalAlloc ::= "alloc" INT;
GlobalAlloc ::= "global" INT;
```

## 全局变量/数组声明

TODO

## 注释和注解

同 [IR2 提案](/ir/llvm-like#注释和注解). 支持的标准注解:

| 名称    | 值      | 含义                              |
| --      | --      | --                                |
| name    | 符号名  | 标记局部变量所对应的符号名称      |
| src     | 文件名  | 标记 IR 文件对应的源代码文件      |
| line    | 行号    | 标记 IR 对应的源代码文件的行数    |
| version | 版本号  | 标记 IR 文件对应的 IR 标准的版本  |

## TODO

咕了.

咕之前放一个 TODO 是一种美德.
