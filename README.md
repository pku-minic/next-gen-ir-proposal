# 关于下一代面向课程的 IR 的提案

该仓库用于记录关于下一代面向课程的 IR 的提案的相关内容.

该仓库持续有效, 你可在此查看 IR/基础设施的最新提案, 同时欢迎[讨论或提交新的提案](#讨论或提交新的提案).

当前提案版本见[现存提案](#现存提案)部分.

## 现存提案

### IR

* [类 LLVM 的 IR (Koopa), v0.4-d](/ir/llvm-like): 更接近工业界的编译器所使用的 IR, 易于开展优化, 但稍显复杂.
* [基于栈的 IR, v0.1-d](/ir/stack): 在实现上更简单, 但相对难被人类理解, 且难于开展优化. 此外, 在这种形式上无法直接进行诸如图着色的复杂寄存器分配策略 (但 n-TOSCA 之类的简单策略是没问题的), 可能和现阶段的课程要求相悖.
* [机器级 IR, v0.1-d](ir/machine): 专为体系结构相关优化设计的 IR, 寄存器分配将在此 IR 上进行.

### 基础设施

* [在线评测系统规范, v0.1-d](/infrastructure/oj): 定义了评测系统的组成, 评测流程, 评测结果判定, 评测状态划分等规范.
* [本地开发环境规范, v0.1-d](/infrastructure/env): 定义了本地开发环境的用户接口.
* [IR 框架规范, v0.1-d](/infrastructure/framework): 定义了 IR 框架的基本结构和 API.

## 讨论或提交新的提案

如需讨论, 请直接提 issue.

如需提交新的提案, 请遵循以下步骤:

1. fork 这个仓库.
2. 在 fork 后的仓库中新建目录, 并添加描述该 IR 细节的 Markdown 文档, 必要时还可在目录中加入其他文件.
3. 向这个仓库发起 pull request, 并等待仓库维护者的 review.

## 提案版本号定义

提案版本号由三部分组成: v`主版本`.`次版本`-`状态`, 例如: `v0.1-d`, `v1.2-r`.

* **主版本**: 不同主版本的提案不保证 API 兼容性.
* **次版本**: 不同次版本的提案在细节上会有差异, 例如更新的提案可能会包含更多功能, 但在 API 上向下兼容.
* **状态**: 提案可能处在两个状态:
  * **d (draft)**: 提案处在草案阶段, 随时可能被修改, 不建议用于正式场合.
  * **r (ratified)**: 提案已被批准, 不会轻易被修改, 可供正式使用.

## 为什么需要新 IR?

现阶段的编译原理课程实践, 使用了两种 IR: 相对更高级的 Eeyore 和相对更低级的 Tigger. 然而根据同学们的反馈, 这两种 IR 并不好用, 或者说, 有些难用. 比如:

* 变量/函数/标号名称固定为 `Txx`, `txx`, `pxx`, `lxx` 和 `f_xx`, 没有任何必要且不利于调试.
* 混杂了变量声明和变量定义的语法, 导致 `var` 声明的变量几乎只能出现在函数开头. 编译器可能需要使用一个额外的 buffer 来存储这些信息, 带来了不必要的处理负担.
* `var` 可以同时定义变量和数组, 但 `var` 定义的符号没有任何类型可言, 导致其定义的变量符号内存储的是变量的值, 而数组符号内存储的是数组的地址, 十分混乱.
* 数组操作按字节寻址. 明明只在生成汇编代码的时候乘个 4 就可以了, 为什么在第一层 IR 生成的时候就要引入这种奇怪的要求?
* 数组长度以字节为单位. 同上, 没必要, 原语言 (MiniC 或 SysY) 本身也只支持 32 位有符号整数.
* Eeyore 和 Tigger 的全局变量/数组初始化语法完全不统一: Eeyore 中的全局变量和数组可以在 `f_main` 之外初始化, 但 Tigger 不支持这么做. 所以大家不得不把所有的初始化过程挪到 `f_main` 之内. 但明明在生成汇编的时候, 是可以避免这种事情的发生的, 例如使用 `.data` 和 `.word`.
* Tigger 里和 load/store 相关的指令多达 7 种, 每种的功能略有差别, 但又有重叠. 然而, 这几种指令所做的事完全可以用两句话和两条指令概括, 这就导致实际使用过程中, 大部分相关指令都极少被用到. 而同学们在读文档的过程中会完完全全被这几种指令干到一脸懵逼黑人问号大呼上当怀疑人生, 然后在群里问那个经典问题: **到底要怎么才能往全局数组写东西?** 毫不客气的说, 设计 Tigger 的人根本不懂设计, 这是典型的 overdesign. 连 RISC-V 都没这么搞, 因为这背离了 RISC 的初衷.
* Tigger 硬编码了 RISC-V 指令系统的寄存器, 同时隐含了 RISC-V 的 ABI, 这不利于支持其他指令系统.
* Tigger 和 RISC-V 汇编十分相似, 查表就可以翻译, 相比而言, Tigger 和 Eeyore 就相去甚远. 把 Eeyore 一步到位翻译到 Tigger, 其实和把 Eeyore 一步到位翻译到 RISC-V 相比, 没有少做太多事情. 这导致很多同学都在生成 Tigger 之前又实现了一种基于虚拟寄存器的新 IR.

上述奇怪的特性导致大家在生成 IR 的过程中总是踩坑. 虽然我认为踩一些坑是好的, 但踩这么多坑实在是难为大家了.

上述特性还导致我们在实现 [MiniVM](https://github.com/pku-minic/MiniVM) 的过程中, 不得不采用一些比较 tricky 的手段去解释这两种 IR, 而且解释的速度还不尽如人意.

此外, 在 Eeyore 和 Tigger 上开展优化也是较为困难的, 这也阻止了同学们在此基础上开发更高性能的编译器 (阻止进一步内卷或许是个好事?).

## IR 和基础设施缺一不可

大部分同学都认为, 编译原理课程实践十分劝退.

> 对, 树洞什么的我们也会看的 (´_っ`).
>
> 助教压力也很大, 怕大家学不好退课, 或者痛骂一顿老师和助教. 所以助教经常在树洞高强度自搜, 收集大家的负面反馈.

1. 这个事情一方面是因为, 第一阶段的任务量, 对于一个新手来说太难了. 大部分同学在没接触过编译器实现的情况下, 一上来就着手完成第一阶段, 需要一些心理建设.

2. 而另一方面在于, 我们并未提供配套的一系列基础设施. 导致同学们必须生成字符串形式的 IR, 在开展实验之前, 也不得不仔细阅读文档, 把文档里的每一个 corner case 都弄清楚. 不这么做的话, 生成的 IR 可能会出现一些语法问题, 或者在运行时出现一些本可以避免的错误, 从而引入不必要的调试负担.

3. 此外, 出于对大家成绩的担心, 我们特意将编译实践分成了三个阶段, 中间使用 Eeyore 和 Tigger 两种 IR 进行衔接. 但这就导致大家如果采用完全分离的实现思路, 就必须为这三个阶段的实现写三个解析器, 工作量激增.

    在此基础上, 很多同学图省事, 直接使用 flex/bison 实现了所有的解析器, 并只实现了 on-the-fly 的编译过程, 没有建立任何数据结构. 而当他们得知, 他们需要把这三个阶段合在一起才能得到有效的性能测试数据的时候, 想必他们也是十分崩溃的.

4. 最后, 编译实践的工具链大多数在 Linux/macOS 上用着比较顺手, 而在 Windows 上就需要折腾一番. 这导致大部分同学都在 Windows 上安装了 Linux 虚拟机, 在虚拟机里完成作业. 而小部分同学试图在 Windows 上搭建工具链, 一部分圆满成功, 另一部分壮烈成仁.

如果我们提供适当的基础设施, 想必上述矛盾会缓和很多.

那么, 基础设施指什么呢? 现阶段我们认为, 基础设施需要包括以下内容:

* **开箱即用的开发环境**: 借助 Docker, 实现几乎无需任何配置, 开箱即用的开发环境. 环境内包含 `git`, `gcc` 等开发工具, 提供可运行 IR 的虚拟机和 RISC-V 工具链, 以及全套调试工具. 开发工具的版本应当和评测系统使用的版本完全一致.
* **简化的本地自动评测脚本**: 开发环境内应该配置一个自动化的本地评测脚本, 用于模拟评测环境的行为, 方便大家在本地调试自己的编译器, 而不必思考自己的编译器在评测平台上究竟发生肾摸事了.
* **用于生成/解析/处理 IR 的代码框架**: 为同学们提供使用课程要求的编程语言 (现阶段是 C/C++, 未来也许可以是 Rust 或其他) 实现的代码框架, 框架提供生成 IR, 解析 IR 和遍历/处理 IR 的功能. 这个框架旨在:
  * **进行错误检查**: 如果生成的 IR 有误, 则编译器的代码直接过不了编译, 或者编译器的代码在运行时会触发异常.
  * **拆分各阶段的实现**: 有了统一的框架, 大家就可以放心的单独实现编译器的各阶段了. 关于怎么把不同的阶段拼起来, 这件事不用整得太明白, 也没必要.
  * **提供更强的可扩展性**: 有了统一的数据结构形式的 IR 表示, 我们就可以抽象各种基于这种数据结构的其他框架, 进可让大家从零开始, 退可让大家代码填空, 课程复杂度也更好控制. 比如实现一个基于 pass 的优化器, 方便大家开展优化工作, 考察大家对编译优化的理解.

我们呼吁, 不仅是北大的编译原理实践课要这么做, 北大的其他计算机类课程, 乃至其他学校的其他计算机类课程, 都应该为同学们提供这种选择.

**不能再让同学们在配环境的一堆烂事上浪费时间了!**
