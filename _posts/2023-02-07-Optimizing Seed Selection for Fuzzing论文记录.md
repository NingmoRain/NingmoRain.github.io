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

我们通过为我们的系统创建一个  a hypothetical fuzzing testbed 来激励我们的研究，称为COVERSET。COVERSET定期监控互联网，下载程序，并对它们进行模糊测试。COVERSET的目标是在有限的时间或预算内最大限度地增加发现的错误数量。由于预算永远是有限的，我们希望做出明智的设计决策，尽可能地使用最优算法。我们该如何着手建立这样一个系统呢？这里给出几个问题：

* 给定数百万、数十亿甚至数万亿的PDF文件，在模糊PDF查看器时应该使用哪些?
* 如何独立于模糊测试调度算法来衡量种子选择技术的质量?
* 我们是否可以为针对特定文件类型的程序的模糊测试提供一个同一个好的种子集?

我们的主要贡献是回答上述问题的技术：

* 我们形式化、实现并测试了一些现有的和新的种子选择算法
* We formalize the notion of ex post facto optimality seed selection and give the first strategy that provides an optimal algorithm even if the bugs found by different seeds are correlated.
* We develop evidence-driven techniques for identifying the quality of a seed selection strategy with respect to an optimal solution.
* 我们在Amazon EC2上使用超过650个CPU天进行了广泛的模糊实验，以获得代表性应用程序的实际情况。总的来说，我们在8个广泛使用的应用程序中发现了240个独特的错误，所有这些错误都在攻击面上(它们通常用于处理不受信任的输入，例如图像、网络文件等)，其中大多数是安全关键型的。

#### 2. Q1：种子选择



 

























































[Optimizing Seed Selection for Fuzzing]:https://ningmorain.github.io/files/sec14_seed_selection.pdf
