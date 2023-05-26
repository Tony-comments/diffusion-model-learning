
# 扩散模型入门

---
- [扩散模型入门](#扩散模型入门)
- [扩散模型入门（一）](#扩散模型入门一)
  - [何为扩散](#何为扩散)
  - [何为扩散模型](#何为扩散模型)
  - [附录：扩散过程的简要数学说明](#附录扩散过程的简要数学说明)
- [扩散模型入门（二）](#扩散模型入门二)
  - [信号及其维度](#信号及其维度)
  - [信号是方差为零的随机变量](#信号是方差为零的随机变量)
  - [信号是均值为零的随机采样](#信号是均值为零的随机采样)

# 扩散模型入门（一）

扩散模型是近年来比较热门的神经网络模型，我认为所谓扩散模型就是对扩散过程进行数学建模，并且能够逆转扩散过程的数学方法。本文将开始从一个初学者的视角尝试理解它的思想和应用。

本文使用的 Toy demo 可见我的在线笔记本

[Diffusion of the curve](https://observablehq.com/@listenzcc/diffusion-of-the-curve)

---


## 何为扩散

所谓扩散就是布朗运动，体现在数学概念上，我们可以将它理解成数据样本点的分布向标准正态分布不断靠拢的过程。这个过程如下图所示，左图中的曲线代表一组高维数据，它拥有连续维度，不同维度的采样值曲线上蓝色点所示，它们的分布如右侧红色点所示。而经过“扩散”之后，采样值的分布逐渐符合标准正态分布，如右图所示。

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%B8%80%EF%BC%89%209bf4745620354c4a819dc19e85867a83/Untitled.png)

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%B8%80%EF%BC%89%209bf4745620354c4a819dc19e85867a83/Untitled%201.png)

## 何为扩散模型

我认为所谓扩散模型就是对上述扩散过程进行数学建模，并且能够逆转扩散过程的数学方法。我觉得这方面介绍得比较清楚的论文是以下这篇，虽然题目中没有扩散的字眼，但“非平衡的热动力学”这个名词非常写意，甚至有些浪漫。

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%B8%80%EF%BC%89%209bf4745620354c4a819dc19e85867a83/Untitled%202.png)

[Deep Unsupervised Learning using Nonequilibrium Thermodynamics](https://arxiv.org/abs/1503.03585)

热力学定律告诫我们，扩散无法避免、熵增无法避免，也就是说宇宙总是向着更加混乱和无序发展的，而维持有序则需要外界不断对某个系统施加能量。那么这个无序的状态可以看作是变量服从正态分布，而有序的低熵状态则代表上述的曲线。它上面的采样值服从未知但确定的分布。

从例子中可以想见，一条有序的曲线经过若干次简单的计算后即可“退回”到正态分布的混沌状态，且这个过程是连续的，或局部可微的。因此，我们同样有理由相信这个过程是“可逆的”，只需要“学会”每一步扩散的微分步骤，就可以在数学上把这个物理上难以维持的过程给反转过来。

为了这个目标所做的一切努力，就是扩散模型，它的优势有四点：（深度网络）模型结构不限、能够实现采样、易与其他概率密度结合、易与样本状态结合。

> 1. extreme flexibility in model structure,
2. exact sampling,
3. easy multiplication with other distributions, e.g. in order to compute a posterior, and
4. the model log likelihood, and the probability of individual states, to be cheaply evaluated.
> 

## 附录：扩散过程的简要数学说明

为了表达方便，我们将采样值（曲线）的分布表示为

$$
x_0 \sim \Psi(x)
$$

它代表一个确定但未知的分布。而我们认为经过一段时间的随机扩散后，新分布属于正态分布

$$
x_T \sim \mathcal{N}(\mu, \Sigma)
$$

为了简单起见，通常假设它服从标准正态分布

$$
\begin{cases}
\mu = 0 \\
\Sigma = I
\end{cases}
$$

因此，扩散过程可以表示成变量连续变化的过程

$$
x_0, x_1, x_2, ..., x_T
$$

直觉上来讲，它可以是一条 Markov 链，其转移概率为

$$
p(x_t) = p(x_{t-1}) \cdot p(x_t|x_{t-1})
$$

连续变化过程如下图所示，希望它看上去有一些动态的感觉。

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%B8%80%EF%BC%89%209bf4745620354c4a819dc19e85867a83/Untitled%203.png)


# 扩散模型入门（二）

前文说到扩散模型是尝试逆转扩散过程的数学方法，那么立即产生的问题就是如何对“扩散”进行可计算的描述。本文尝试从两个角度解读我们日常观测到的信号，相信它们不仅有助于理解随机变量，也有助于后续引入扩散的计算方法。在扩散模型中，我更加倾向于将红线理解成对均值为零的随机变量进行的一次采样，它背后的逻辑是“随机性先于一切而存在，而我们观测到的信号只是对某个高维的随机变量进行了一次采样”。

本系列相关代码整合在 Github 仓库

[https://github.com/listenzcc/diffusion-model-learning](https://github.com/listenzcc/diffusion-model-learning)

---

## 信号及其维度

信号是某个客观事物的直观表达，我们可以用高维向量的观点去理解它。假设我们对一个连续的函数进行采样，那么每个采样值就是它的维度。下图是一个例子，左侧是一张 $10 \times 10$ 的图（虽然它没能实际意义），右侧中红线是图中的每个像素转换成一条向量 $\hat{x}$，而它背后的蓝白色曲线就是它加上一些噪声的例子，为了方便起见，我们将它们统称为“信号”，该信号也是 $100$ 维的向量。

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%BA%8C%EF%BC%89%2029284216d2864644b67c667a8e16451c/Untitled.png)

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%BA%8C%EF%BC%89%2029284216d2864644b67c667a8e16451c/Untitled%201.png)

接下来，我们关注蓝色曲线，这是典型的“确定值加噪声的形式”，它们满足正态分布

$$
\begin{cases}
X \sim \mathcal{N}(\mu, \Sigma)\\
\mu = \hat{x}\\
\Sigma = I
\end{cases}
$$

易知，每条蓝色曲线出现的概率为

$$
p(x) = \mathcal{N}(x; \mu, \Sigma)
$$

下面我们开始考虑一个问题，那就是红色的线是什么？这个问题十分重要，因为我们如何理解它，决定了我们如何对它进行扩散和建立对应的扩散模型。我现在有两种理解方式，一是将红线理解成方差为零的随机变量，二是将红线理解成对均值为零的随机变量进行的一次采样。

## 信号是方差为零的随机变量

将红线理解成方差为零的随机变量是非常直观的想法，它背后的逻辑是

> 信号先于随机性而存在，以均值的形式存在，而观测这个信号得到的值是均值与噪声相加得到的采样值。
> 

这个过程如下图所示，随着噪声方差越来越小，信号的“不确定性”也越来越小，直到减少到零时得到完全确定的信号。右侧的 colorbar 代表该颜色的采样值曲线与红线之间的欧氏距离。这种思想无疑是有效的，但其有效性不会突破它的适用范围，也就是说它适用于解决那些“信号先于随机性存在的问题”，**只要我们能够把方差降下来，那么感兴趣的信号就会自然浮现出来**。比如通过多次取平均的方式减少系统误差的方法就是典型应用之一

$$
\begin{cases}
\bar{X} = \frac{1}{n} \sum_{i=1}^{n} X \\
Var(\frac{X}{n}) = \frac{Var(X)}{n^2}
\end{cases}
$$

$$
Var(\bar{X}) = \frac{Var(X)}{n}
\rightarrow
\lim_{n \rightarrow \infty} Var(\bar{X}) = 0
$$

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%BA%8C%EF%BC%89%2029284216d2864644b67c667a8e16451c/Untitled%202.png)

## 信号是均值为零的随机采样

另一个观点是将红线理解成对均值为零的随机变量进行的一次采样，它背后的逻辑是

> 随机性先于一切而存在，而我们观测到的信号只是对某个高维的随机变量进行了一次采样。
> 

当然，这里我们需要进一步分析这个高维空间长什么样子，以及如何从概率密度的角度研究我们为什么能够，或者说已经观察到这组采样结果。

![Untitled](%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B%E5%85%A5%E9%97%A8%EF%BC%88%E4%BA%8C%EF%BC%89%2029284216d2864644b67c667a8e16451c/Untitled%203.png)