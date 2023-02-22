---
layout: post
title: Symbolic_Execution_Survey论文记录
categories: [论文记录]
description: Symbolic_Execution_Survey论文记录
keywords: 论文记录
---

### [A Survey of Symbolic Execution Techniques][]

[ppt][]

#### 摘要

许多安全和软件测试应用程序需要检查程序的某些属性是否适用于任何可能的使用场景。例如，一个识别软件漏洞的工具可能需要排除任何绕过程序身份验证的后门的存在。一种方法是使用不同的、可能是随机的输入来测试程序。由于后门可能只会被非常特定的程序工作负载击中，因此自动探索可能的输入空间是至关重要的。符号执行为这个问题提供了一个优雅的解决方案，它系统地同时探索许多可能的执行路径，而不一定需要具体的输入。该技术不是采用完全指定的输入值，而是将它们抽象为符号，并借助于约束求解器来构建实际的实例，从而导致属性违反。符号执行已经在过去40年开发的几十个工具中孵化出来，导致了许多著名的软件可靠性应用的重大实际突破。本调查的目标是提供该领域的主要想法、挑战和解决方案的概述，将它们提炼出来供广大受众使用。

 #### 1. 介绍

普通的测试只允许输入一个 input，从而探索到有限的路径；而符号执行不是使用具体的数值，而是使用 符号 来代替，从而可以探索到可能的分支。

一个 符号执行引擎（symbolic execution engine）在过程中需要维护：

* 一阶布尔公式，描述沿着该路径的分支所需要满足的条件
* 将变量映射到符号表达式或值的符号存储器（symbolic memory store）。

遇到分支则会更新布尔公式（进入分支的条件），而赋值语句更新符号存储器。

在任一个程序点，一般使用基于可满足性模理论(satisfiability modulo theories，SMT)的求解器作为模型检查器（model checker），用于验证探索路径是否存在任何违反属性的情况，以及路径本身的公式是否可以通过对程序的符号参数的一些具体值的分配来满足。

##### 1.1 一个小例子

考虑下图中的代码，假设我们想要找到哪些 input 可以使得函数触发第八行的断言。因为每个4字节的输入参数可以有多达$$2^{32}$$个不同的整数值，这会导致我们随机探索的空间过大。但是符号执行可以解决这个问题。

<img src="D:\Typora图像\image-20230214094642497.png" alt="image-20230214094642497" style="zoom: 33%;" />

更详细地说，每个不能通过代码的静态分析来确定的值，例如函数的实际参数或从流中读取数据的系统调用的结果，都由符号$$α_i$$表示。在任何时候，符号执行引擎保持一个状态$$(stmt， σ， π)$$，其中:

* stmt 是下一个要求解的语句
* σ 是一个符号存储，它将 程序变量 与 具体值 的表 达式 或 符号值$$α_i$$ 关联起来。
* π 表示路径约束，是一个布尔公式，它表达了符号 $$α_i$$ 上的一组假设，这些假设是在执行到 stmt 时所采取的分支。在分析最开始时，$$π = true$$。

<img src="D:\Typora图像\image-20230214095402916.png" alt="image-20230214095402916" style="zoom: 67%;" />

##### 1.2 符号执行的挑战

穷尽式符号执行不太可能扩展到小型应用程序之外。符号执行在真实应用时可能面临一些挑战

* 存储：符号执行引擎 如何处理**指针、数组或其他复杂对象**？操作指针和数据结构的代码不仅会产生符号存储的数据，还会产生由符号表达式描述的地址。
* 环境：引擎如何处理跨软件栈的交互？**对库和系统代码的调用可能会导致副作用**（side effects）。
* 状态空间爆炸：符号执行如何处理路径爆炸？像循环这样的语言构造可能会成倍地增加执行状态的数量。因此，符号执行引擎不太可能在合理的时间内彻底地探索所有可能的状态。
* 约束求解：约束求解器在实践中可以做什么？SMT求解器可以扩展到数百个变量的约束的复杂组合。然而，非线性算法等结构严重影响了效率

##### 1.4 文章组织

在第2节中，我们将讨论符号执行引擎的总体原则和计算策略

第3节至第6节讨论了我们在第1.2节中列出的主要挑战

第7节讨论了如何应用其他领域的最新进展来增强符号执行技术

#### 2. 符号执行引擎

##### 2.1 混合符号执行与具体执行

虽然建模所有可能的运行可以进行非常精准分析，但在实践中通常是不可行的，特别是在现实世界的软件中。

经典符号执行的一个主要限制是，无法处理的路径约束 是无法探索可行执行的。例如执行器无法追踪的外部代码、非线性算术等复杂约束。由于在约束求解上花费的时间是引擎的主要性能障碍，所以可解决性和效率需要一定的平衡。此外，实际的程序通常不是自包含的，实现一个能够静态分析整个软件堆栈的符号引擎相当具有挑战性，因为在执行期间很难准确评估可能产生的副作用。

解决这些问题并使象征性执行在实践中可行的一个基本想法是将具体执行和象征性执行混合在一起:这被称为concolic 执行

###### 动态符号执行（DSE）

动态符号执行(dynamic symbolic execution, DSE)[51]可以有效地缓解上述问题。

除了 符号存储σ 和 路径约束π ，执行引擎还维护一个具体的存储 $$σ_c$$。在选择任意输入之后，它通过同时更新两个存储和路径约束来具体和符号地执行程序。每当具体执行进入一个分支时，符号执行就会指向同一个分支，并且从分支条件中提取的约束会添加到当前的路径约束集。

为了探索不同的路径，可以对其他分支给出的路径条件进行否定，并调用SMT求解器来为新约束找到一个令人满意的赋值，即生成一个新的输入。

尽管DSE使用具体的输入来驱动符号执行走向特定的路径，但每当需要探索新的路径时，它仍然需要选择一个分支来否定。还要注意，每次具体执行都可能会添加必须访问的新分支。由于所有执行的具体执行中未执行的分支的集合可能非常大，因此采用有效的搜索启发式算法(2.2节)可以发挥至关重要的作用。

引擎可以利用在具体运行过程中维护的符号信息来获取新的输入并探索新的路径。下一个示例显示DSE如何使用 concolic 引擎处理没有 符号化跟踪 的外部代码调用。使用具体值来帮助约束求解将在第6节中讨论。

<img src="D:\Typora图像\image-20230214102053897.png" alt="image-20230214102053897" style="zoom: 67%;" />

对a而言，分支条件与 bar() 无关，所以比较正常。

对b而言，分支与a有关，a是x经过bar()处理的结果，因为没有跟踪，所以没办法求解，称为假阴性(即错过的路径)

对c而言，会发现就算对分支求解之后，仍然无法进入分支。这种情况被称为路径发散（path divergence）

###### 选择性符号执行（S2E）

Selective Symbolic Execution $$S^2E$$[29]采用了一种不同的方法，也就是只全面探索软件堆栈的一些组件，而不关心其他组件

假设一个函数A调用了一个函数B，并且在调用点改变了执行模式。出现了两种情况:(1)From concrete to symbolic and back：当前函数A正在具体执行，B也将被具体执行，并且将B的具体结果返回给A。同时，B的参数也被符号化，并且对B进行完全的符号执行。(2)From symbolic to concrete and back：函数A正在符号执行，B的参数具体化，B被具体执行，最后函数A继续符号执行。

##### 2.2 路径选择

由于枚举程序的所有路径可能会非常昂贵，因此在许多与测试和调试相关的软件工程活动中，通过首先查看最有希望的路径来优先搜索。找到一个普遍的最优策略仍然是一个悬而未决的问题。

最常用的策略是深度优先搜索(DFS)和广度优先搜索(BFS)，前者在回溯到最深的未探索分支之前尽可能地扩展路径，后者则并行扩展所有路径。当内存使用非常昂贵，但被包含循环和递归调用的路径所阻碍时，通常采用DFS。BFS允许引擎快速探索不同的路径，及早发现有趣的行为。另一种流行的策略是随机路径选择，它已经经过了多种改进。例如，KLEE[20]根据路径的长度和分支度为路径分配概率:它倾向于探索次数较少的路径，防止因循环和其他路径爆炸因素造成的饥饿。

一些工作，如EXE [21]， KLEE [20]， Mayhem[25]和S2E[29]，讨论了以最大化代码覆盖率为目标的启发式算法。例如，KLEE[20]中讨论的覆盖优化搜索为每个状态计算一个权重，稍后用于随机选择状态。权重通过考虑最近未覆盖指令的距离、新代码最近是否被该状态覆盖以及该状态的调用栈来获得。

[71]中提出的子路径引导搜索，它试图通过选择控制流图中探索次数较少的子路径来探索程序中探索次数较少的部分。这是通过维护已探索子路径的频率分布来实现的，其中子路径被定义为一条完整路径的长度为n的连续子序列。有趣的是，值n在使用这种启发式的符号引擎实现的代码覆盖率方面起着至关重要的作用，没有任何特定的值被证明是普遍最优的。

其他搜索启发式方法试图根据目标确定可能导致感兴趣状态的路径的优先级。例如，AEG[8]就引入了两种这样的策略。漏洞路径优先策略选择的路径的过去状态包含小但不可利用的漏洞。直觉是，如果一条路径包含一些小错误，那么它很可能没有经过适当的测试。因此，未来的状态很有可能包含有趣的、有望被利用的bug。循环穷举策略探索访问循环的路径。这种方法的灵感来自于实际观察，循环中的常见编程错误可能会导致缓冲区溢出或其他与内存相关的错误。为了找到可利用的漏洞，Mayhem[25]反而优先考虑标识了对符号地址的内存访问或检测到符号指令指针的路径。

[118]提出了一种新的动态符号执行方法，以自动找到满足常规属性的程序路径，即可以由有限状态机(FSM)表示的属性(如文件使用或内存安全)。动态符号执行由有限状态机(FSM)指导，以便首先探索最可能满足性质的执行路径分支。该方法采用静态分析和动态分析相结合的方法计算探索路径的优先级:符号执行过程中动态计算当前路径已到达的FSM状态，静态计算未来状态。如果这两个集合的交集不为空，则可能存在一条满足该性质的路径。

##### 2.3 反向符号执行(SBE)

反向符号执行(SBE)[26,40]是符号执行的一种变体，其中探索从目标点反向到程序的入口点。因此，分析是以与规范(正向)符号执行相反的方向进行的。这种方法的主要目的通常是识别一个可以触发执行特定代码行的测试输入实例(例如，assert或throw语句)。当开发人员对程序进行调试或回归测试时，这可能非常有用。当探索从目标开始时，沿着遍历过程中遇到的分支收集路径约束。SBE引擎可以同时探索多条路径，类似于正向符号执行，路径会定期检查其可行性。当路径条件被证明不可满足时，引擎丢弃路径并返回。

[72]讨论了SBE的一个变体，称为调用链反向符号执行(CCBSE)。该技术首先确定函数中目标行所在的有效路径。当找到路径时，引擎移动到包含目标点的函数的调用者之一，并尝试重建从调用者的入口点到目标点的有效路径。该过程递归重复，直到从程序的主函数重构出一条有效的路径为止。与传统的SBE相比，主要的区别在于，尽管CCBSE从目标点向后遵循调用链，但在每个函数内部，探索都像传统的符号执行那样进行。

与CCBSE一样，SBE中的逆向探索的一个重要需求是过程间控制流图的可用性，它提供了一个完整的程序控制流图，并使得确定探索过程中函数的调用位置成为可能。不幸的是，构建这样的图在实践中可能非常具有挑战性。此外，一个函数可能有许多可能的调用点，这使得SBE执行的探索仍然非常昂贵。另一方面，如果从相反的方向收集约束，则会产生一些实际的优势。我们将在第6节进一步讨论这些好处。

##### 2.4 符号执行器的设计原则

试图在一次运行中同时执行多个路径的符号执行器(也称为在线执行器)在每个依赖输入的分支上克隆执行状态。给出了KLEE [20]， AEG [8]， s2e[29]中的例子。这些引擎不会重新执行之前的指令，因此避免了工作重复。然而，许多活动状态需要保存在内存中，内存消耗可能很大，这可能会阻碍进程的进行。

像混合执行(concolic execution)那样，一次只能推断一条路径，这是所谓的离线执行器程序(如SAGE[52])所采用的方法。相对于在线的执行器进程来说，独立地运行每条路径可以降低内存消耗，并且可以立即重用之前运行的分析结果。另一方面，工作在很大程度上是重复的，因为每次运行通常都会从头开始重新执行程序。

#### 3. 存储模型

内存模型主要解决的问题是如何支持带有指针和数组的程序。因为这代表着我们不仅要将 变量映射到符号表达式和具体值，也要将内存地址映射到符号表达式和具体值。

##### 3.1 全符号存储

###### 状态分叉

如果一个操作读取或写入一个符号地址，则通过考虑该操作可能产生的所有可能的状态来划分状态。路径约束会针对每个分叉状态进行相应的更新。也就是对于所有可能的地址做一个分支。

###### if-then-else公式

设计一个$$ite(a,b,c)$$公式，如果条件a是真的就执行b，否则执行c。通过这个公式可以判断将数组结果符号化。$$a[0] = ite(α_i=0,5,0),a[1]=ite(α_i=1,5,0)$$，也就是说，如果index $$α_i$$ 等于0就将a[0]赋值为5，否则赋值原来的结果。

因为全符号存储比较精确，所以能够适用于比较小且确定的数组，如果是动态分配的地址就没法分析了，所以提出了以下方向：

* 以紧凑的形式表示内存。[32]将符号地址表达式映射到数据。查询被卸载到高效的 paged interval tree 实现，以确定哪些存储的数据可能被内存读取操作引用。
* 用准确性换效率。通过将符号指针替换为具体地址，将符号探索限定为执行状态的子集
* 堆模型。将探索限制在指针被限制为空或指向先前堆分配的对象的状态，而不是任何通用内存位置

##### 3.2 地址具体化

地址具体化将指针具体化到单个特定地址。DART和CUTE，它们通过将 T* 类型的引用具体化为NULL或新分配的sizeof(T)字节对象的地址来处理内存初始化。DART随机做出选择，而CUTE首先尝试NULL，然后在随后的执行中使用一个具体的地址。

 ##### 3.3 部分存储模型

全符号内存每个状态指向一个具体的地址，状态太多；地址具体化所有状态都指向同一个地址，精确度损失太大。所以Mayhem提出每个状态指向一个范围内所有指针值，从而取得两种方法中的平衡。

##### 3.4 Lazy Initialization

[66]为高级面向对象语言构造提出了符号执行技术。

在执行过程中首次访问一个对象时初始化它们。

当访问一个未初始化的对象时，算法将当前状态与三种不同的堆配置进行分叉，其中字段初始化为:(1)null，(2)对具有所有符号属性的新对象的引用，以及(3)先前引入的所需类型的具体对象。从而利用这样的状态可以确定一个具体的对象。

#### 4. 与环境交互

大多数程序并不是自包含的，宽泛点来说就是程序之外的我们无法进行分析的接口所带来的影响的问题。

**系统环境**：早期的代表工作DART [51]， CUTE[91]都是使用具体的参数来实际执行外部调用，然后使用具体的值进行符号执行。带来的问题就是极大的限制了可以探索的路径。

另一个方向是创建抽象模型来捕获这些交互。比如KLEE对文件系统进行建模，也可以对系统调用进行建模。

程序通过执行系统调用、接收信号等与环境交互。当执行到达不受符号执行工具控制的组件(例如，内核或库)时，可能会出现一致性问题。考虑下面的例子:

```
int main()
{
  FILE *fp = fopen("doc.txt");
  ...
  if (condition) {
    fputs("some data", fp);
  } else {
    fputs("some other data", fp);
  }
  ...
  data = fgets(..., fp);
}
```

这个程序打开一个文件，并根据某些条件，将不同类型的数据写入该文件。然后再读回写入的数据。理论上，符号执行将在第5行分两条路径，从那里开始的每条路径都有自己的文件副本。因此，第11行的语句将返回与第5行的“condition”值一致的数据。实际上，文件操作是在内核中作为系统调用实现的，不受符号执行工具的控制。应对这一挑战的主要方法是:

**直接执行对环境的调用**。这种方法的优点是实现起来很简单。缺点是这种调用的副作用将破坏符号执行引擎管理的所有状态。在上面的例子中，第11行的指令将根据状态的顺序返回“some datasome other data”或“some other datasome data”。

**环境建模。**在这种情况下，引擎使用一个模型来模拟系统调用，并将所有副作用保存在每个状态存储中。这样做的好处是，在象征性地执行与环境交互的程序时，可以得到正确的结果。缺点是需要实现和维护许多潜在的复杂的系统调用模型。

**分叉整个系统状态。**基于虚拟机的符号执行工具通过分叉整个VM状态来解决环境问题。例如，在S2E中，每个状态都是一个独立的虚拟机快照，可以单独执行。这种方法减少了编写和维护复杂模型的需要，并允许象征性地执行几乎任何程序二进制。但占用内存较多(可能会导致虚拟机快照较大)。

#### 5. 路径爆炸

路径爆炸的主要来源是循环和函数调用。循环的每次迭代都可以被视为if-goto语句，导致执行树中的条件分支，而且有时候循环的范围可能是无限的。最简单的办法就是假设循环都是固定次数的，比如两次，但是这样对精度的损失太大了。

##### 5.1 不可达路径剪枝

![image-20230215150424934](https://ningmo.oss-cn-beijing.aliyuncs.com/img/image-20230215150424934.png)

在每一个分支出都调用求解器求解，如果布尔公式是不可满足的，那就没有必要继续探索了，这样的剪枝并不影响分析的准确性。但是这会增加SMT求解器的负担，毕竟求解效率是符号执行的主要瓶颈。

##### 5.2 函数 和 循环 摘要

当一个代码片段(无论是函数还是循环体)被遍历几次时，符号执行器可以构建其执行的摘要，以供后续重用。

**Function Summaries.**:函数可能被多次执行，如果在同一上下文中，可以尝试重用先前的分析结果。在每次具体执行中收集的输入和输出的约束关联，在之后调用API时可以重用相关结果。

**Loop Summaries.**:和上面的方案相同，循环也可以获取摘要，循环摘要使用在符号执行期间通过推理循环条件和符号变量之间的依赖关系动态计算的前置条件和后置条件。

##### 5.3 路径包含与路径等价

**插值：**从没有显示所需属性的程序路径中获得属性，从而防止探索同样不能满足它的类似路径。

克雷格插值[34]允许决定公式的哪些信息与属性相关。假设在某些逻辑中蕴涵P→Q成立，可以构造一个I，使得P→I和I→Q是有效的，并且I中的每一个非逻辑符号都出现在P和Q中。插值通常用于程序验证:给出一个不可满足公式P∧Q的反驳证明，可以构造一个反插值I，使得P→I是有效的，I∧Q是不可满足的。也就是说我们可以提前获得不可满足的路径，从而在遇到这种路径时不进入。

包含与抽象。在[6]中描述了一种用于符号状态的双重包容检查技术，提出了一种通过图遍历来匹配堆配置的算法，同时使用现成的求解器来推理标量数据的包含。

路径分区。控制流和数据流的依赖关系分析暴露了可以在探索过程中使用的偶然关系，以过滤掉无法显示其他程序行为的路径。[74]在不干扰的块中划分concolic执行的输入，象征性地探索每个块，而其他块则保持固定的具体值。当两个输入共同影响一个语句或由控制或数据依赖项链接的语句时，就会发生两个输入的干扰。[84]重点关注输出，如果两条路径与程序输出有相同的相关片，则将它们放在同一个分区中。一个相关的片是动态数据和控件依赖关系的传递闭包，也是潜在依赖关系的传递闭包，这些依赖关系涉及不执行而影响输出的语句。[109]还通过为单个语句构建相关切片来探索与输出无关的错误，捕捉它们是如何从符号输入中计算的。依赖分析可以有效地检查片的等价性，当其所有语句实例的片都被前面的路径共同覆盖时，就认为路径是冗余的。



















[A Survey of Symbolic Execution Techniques]:https://ningmorain.github.io/files/Survey__Symbolic_Execution.pd
[ppt]:https://ningmorain.github.io/files/SymbolicExecution.pptx