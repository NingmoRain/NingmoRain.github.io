---
layout: post
title: Optimizing Seed Selection for Fuzzing
categories: [论文记录]
description: Optimizing Seed Selection for Fuzzing论文记录
keywords: 论文记录
---



### [Optimizing Seed Selection for Fuzzing][]

#### 摘要

随机变异格式良好的程序输入或简单的模糊测试是一种高效且广泛使用的软件缺陷发现策略。除了让模糊测试器找到bug之外，在理解如何正确进行模糊测试的科学方面，几乎没有系统的努力。本文专注于如何从数学上形式化和推理模糊测试中的一个关键方面:如何最好地选择种子文件，以最大化在模糊测试过程中发现的bug总数。设计和评估了6种不同的算法，在亚马逊弹性计算云(EC2)上使用了650多个CPU天来提供真实数据。总的来说，我们在8个应用中发现了240个bug，这表明所选择的算法可以大大增加发现的bug数量。我们还表明，在Peach中发现的当前种子选择策略可能不会比随机选择种子更好。我们公开了我们的数据集和代码。

#### 1.介绍

软件测试很重要，Fuzzing 现在应用很多。Fuzzing 吸引人的一个原因是它相对简单和快速地开始工作。文章详细的区分不同的 Fuzzer ，只是从中挑选了一个典型的Fuzzer——BFF。为了评估种子选择策略，我们使用了流行的调度算法，如round-robin算法，以及the best possible (optimal) scheduling。

**我们通过为我们的系统创建一个  a hypothetical fuzzing testbed 来激励我们的研究，称为COVERSET。COVERSET定期监控互联网，下载程序，并对它们进行模糊测试。COVERSET的目标是在有限的时间或预算内最大限度地增加发现的错误数量。由于预算永远是有限的，我们希望做出明智的设计决策，尽可能地使用最优算法。**我们该如何着手建立这样一个系统呢？这里给出几个问题：

* 给定数百万、数十亿甚至数万亿的PDF文件，在模糊PDF查看器时应该使用哪些?
* 如何独立于模糊测试调度算法来衡量种子选择技术的质量?
* 我们是否可以为针对特定文件类型的程序的模糊测试提供一个同一个好的种子集?

我们的主要贡献是回答上述问题的技术：

* 我们形式化、实现并测试了一些现有的和新的种子选择算法
* We formalize the notion of ex post facto optimality seed selection and give the first strategy that provides an optimal algorithm even if the bugs found by different seeds are correlated.
* We develop evidence-driven techniques for identifying the quality of a seed selection strategy with respect to an optimal solution.
* 我们在Amazon EC2上使用超过650个CPU天进行了广泛的模糊实验，以获得代表性应用程序的实际情况。总的来说，我们在8个广泛使用的应用程序中发现了240个独特的错误，所有这些错误都在攻击面上(它们通常用于处理不受信任的输入，例如图像、网络文件等)，其中大多数是安全关键型的。

#### 2. Q1：种子选择

对每个程序进行足够时间的模糊处理以使其在所有种子文件中都有效，这在计算上是非常昂贵的。此外，种子文件集在模糊过程中引起的行为通常是重复的

 Peach建议使用执行的代码覆盖率作为种子选择策略，其直观的想法是，许多种子文件可能执行相同的代码块，而这些种子可能产生相同的错误。以前的工作已经显示了覆盖率和发现的bug之间的相关性，但还没有在许多方法之间进行比较研究，也没有研究如何衡量最优性。

**SCP（set cover problem）**：

* 假设当前存在一个种子文件的集合 $\mathbb F=\{S_1,...,S_n\}$
* 其中每个种子能够覆盖一定的代码 $S_i=\{codeline\}$
* 假设所有种子文件能够覆盖的代码行是$X=\bigcup_{S\in \mathbb F} S$
* 如果存在一个种子文件子集合 $\mathbb C \subseteq \mathbb F$，其覆盖的代码行是 $\bigcup_{S\in \mathbb C} S=X$
* 则称 $\mathbb C$ 为一个集合覆盖

**MSCP（minimal set cover problem）**：最小化集合覆盖$\mathbb C$ ，称为minset。minset可能是不唯一的，可能有多个满足条件的minset

**WMSCP（weighted minimal set cover problem）**：为每一个种子文件设置一个$Cost()$函数，寻找cost最小的minset。

MSCP和WMSCP都可以扩展可选参数k(分别形成k- scp和k- wscp)，指定返回解的最大大小，也就是寻找在指定大小的集合下，尽可能最大化覆盖，并且尽可能最小化cost。

##### 种子选择算法

* Peach Set：基于贪婪的集合覆盖

  <img src="D:\Typora图像\image-20230207143810221.png" alt="image-20230207143810221" style="zoom: 50%;" />

* Random Set：随机选择k个种子文件

* Hot Set：每个种子文件单独测试五分钟，记录每个种子文件发现的bug数，取效果最好的k个种子文件

* Unweight Minset：选取k个种子文件，要求达到最高的覆盖范围

* Time Minset：选取k个种子文件，要求达到最高的覆盖范围，并且尽可能少的运行时间

* Size Minset：选取k个种子文件，要求达到最高的覆盖范围，并且尽可能小的文件大小

##### 具体问题

假设1：给定相同的大小参数k, MINSET算法比RANDOM SET算法能发现更多的bug。

假设2：计算给定应用程序和种子文件集的MINSET，然后开始模糊测试，比直接进行模糊测试能够发现更多的bug

假设3：minset在不同的程序上同时有效

假设4：使用minset测试能够比使用所有种子文件进行测试有更好的结果。

#### 3. RQ2：测量选择质量

在我们的方法中，主要的想法是衡量在特定种子子集中发现的bug的最佳情况。最佳情况提供了任意调度算法的上界，而不是特定调度算法的上界。

为了计算最优情况，对每一个种子文件 $s_i$​ 进行时间 t 的模糊测试，记录这个过程中能够发现的bug。利用模糊测试的结果计算事后的最优搜索策略，以最大限度地增加发现的bug数量。我们发现寻找最优调度算法本质上是一个整数规划问题，我们用整数线性规划(ILP)的方法来求解最优种子调度问题。

##### 形式化

* 假设模糊测试的时间限制是 $t_{thres}$
* 对每一个种子文件单独进行模糊测试，记录过程中发现的bug $Fuzz(\{S_i\},t_{thres})=\{(b_1,S_i,t_1),...,(b_k,S_i,t_k)\}$
  * $b_k$代表发现的第 k 个bug的内容
  * $t_k$代表发现第k个bug的时间戳

我们准备将具有固定时间预算的最优调度描述为ILP最大化问题

<img src="D:\Typora图像\image-20230207150910037.png" alt="image-20230207150910037" style="zoom:50%;" />

约束(2)确保调度考虑发现的崩溃的顺序。特别地，如果找到了种子的第j个崩溃，则必须同时找到所有之前的崩溃。

约束(3)确保找到所有崩溃的时间不会超过我们的时间预算。

约束(4)表示，如果找到了一个崩溃，就找到了它对应的bug(基于stack-hash)

约束(5)保证，如果找到了一个bug，就至少找到了一个触发这个bug的崩溃。



除此之外，文章也尝试反向来使用求解，即对一个确定的调度 Round-Robin，能够反向选择最优的种子集合。

#### 4. Q3：种子文件可移植性

为单个应用P1预先计算好的种子集可能会耗费大量时间。COVERSET认为最小化总体成本的一种方法是找到一组“好的”种子，并在一个应用程序到另一个应用程序中重用它们。有几个理由相信这可能会奏效。

原因之一是大多数程序只依赖少数库来处理PDF、图像和文本。例如，如果应用程序P1和P2都链接到poppler PDF库，那么两个应用程序都可能在相同的输入上崩溃。然而，共享库通常很容易检测，而且这种情况可能没有意义。假设P1和P2都有独立的实现，例如P1使用poppler, P2使用GhostScript图形库。在类似的PDF上，P1和P2可能会崩溃的一个原因是，PDF标准中有一些难以正确实现的部分，因此它们都很可能出错。然而，人们可以推测出许多原因，这些应用程序中的bug是独立的。据我们所知，当通过模糊测试发现漏洞时，还没有系统的调查来解决这个问题。



















































[Optimizing Seed Selection for Fuzzing]:https://ningmorain.github.io/files/sec14_seed_selection.pdf
