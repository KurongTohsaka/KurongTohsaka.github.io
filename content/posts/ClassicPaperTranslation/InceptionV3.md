---
title: "Inception V3"
date: 2024-06-26
tags: ["NN"]
categories: ["ClassicPaperTranslation"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
searchHidden: false
---

# Inception V3

## Abstract

对许多任务而言，卷积网络是目前最新的计算机视觉解决方案的核心。从2014年开始，深度卷积网络开始变成主流，在各种基准数据集上都取得了实质性成果。对于大多数任务而言，虽然增加的模型大小和计算成本都趋向于转化为直接的质量收益（只要提供足够的标注数据去训练），但计算效率和低参数计数仍是各种应用场景的限制因素，例如移动视觉和大数据场景。目前，我们正在探索增大网络的方法，目标是通过适当的分解卷积和积极的正则化来尽可能地有效利用增加的计算。我们在ILSVRC 2012分类挑战赛的验证集上评估了我们的方法，结果证明我们的方法超过了目前最先进的方法并取得了实质性收益：对于单一框架评估错误率为：`21.2% top-1`和`5.6% top-5`，使用的网络计算代价为每次推断需要进行50亿次乘加运算并使用不到2500万的参数。通过四个模型组合和多次评估，我们报告了`3.5% top-5`和`17.3% top-1`的错误率。



## 1. Introduction

从2012年Krizhevsky等人[9]赢得了ImageNet竞赛[16]起，他们的网络“AlexNet”已经成功了应用到了许多计算机视觉任务中，例如目标检测[5]，分割[12]，行人姿势评估[22]，视频分类[8]，目标跟踪[23]和超分辨率[3]。

这些成功推动了一个新研究领域，这个领域主要专注于寻找更高效运行的卷积神经网络。从2014年开始，通过利用更深更宽的网络，网络架构的质量得到了明显改善。VGGNet[18]和GoogLeNet[20]在2014 ILSVRC [16]分类挑战上取得了类似的高性能。一个有趣的发现是在分类性能上的收益趋向于转换成各种应用领域上的显著质量收益。这意味着深度卷积架构上的架构改进可以用来改善大多数越来越多地依赖于高质量、可学习视觉特征的其它计算机视觉任务的性能。网络质量的改善也导致了卷积网络在新领域的应用，在AlexNet特征不能与手工精心设计的解决方案竞争的情况下，例如，检测时的候选区域生成[4]。

尽管VGGNet[18]具有架构简洁的强有力特性，但它的成本很高：评估网络需要大量的计算。另一方面，GoogLeNet[20]的Inception架构也被设计为在内存和计算预算严格限制的情况下也能表现良好。例如，GoogleNet只使用了500万参数，与其前身AlexNet相比减少了12倍，AlexNet使用了6000万参数。此外，VGGNet使用了比AlexNet大约多3倍的参数。

Inception的计算成本也远低于VGGNet或其更高性能的后继者[6]。这使得可以在大数据场景中[17]，[13]，在大量数据需要以合理成本处理的情况下或在内存或计算能力固有地受限情况下，利用Inception网络变得可行，例如在移动视觉设定中。通过应用针对内存使用的专门解决方案[2]，[15]或通过计算技巧优化某些操作的执行[10]，可以减轻部分这些问题。但是这些方法增加了额外的复杂性。此外，这些方法也可以应用于优化Inception架构，再次扩大效率差距。

然而，Inception架构的复杂性使得更难以对网络进行更改。如果单纯地放大架构，大部分的计算收益可能会立即丢失。此外，[20]并没有提供关于导致GoogLeNet架构的各种设计决策的贡献因素的明确描述。这使得它更难以在适应新用例的同时保持其效率。例如，如果认为有必要增加一些Inception模型的能力，将滤波器组大小的数量加倍的简单变换将导致计算成本和参数数量增加4倍。这在许多实际情况下可能会被证明是禁止或不合理的，尤其是在相关收益适中的情况下。在本文中，我们从描述一些一般原则和优化思想开始，对于以有效的方式扩展卷积网络来说，这被证实是有用的。虽然我们的原则不局限于Inception类型的网络，但是在这种情况下，它们更容易观察，因为Inception类型构建块的通用结构足够灵活，可以自然地合并这些约束。这通过大量使用降维和Inception模块的并行结构来实现，这允许减轻结构变化对邻近组件的影响。但是，对于这样做需要谨慎，因为应该遵守一些指导原则来保持模型的高质量。



## 2. General Design Principles

这里我们将介绍一些具有卷积网络的、具有各种架构选择的、基于大规模实验的设计原则。在这一点上，以下原则的效用是推测性的，另外将来的实验证据将对于评估其准确性和有效领域是必要的。然而，严重偏移这些原则往往会导致网络质量的恶化，修正检测到的这些偏差状况通常会导致改进的架构。

1. **避免表征瓶颈，尤其是在网络的前面**。前馈网络可以由从输入层到分类器或回归器的非循环图表示。这为信息流定义了一个明确的方向，对于分离输入输出的任何切口，可以访问通过切口的信息量。应该避免极端压缩的瓶颈。**一般来说，在达到用于着手任务的最终表示之前，表示大小应该从输入到输出缓慢减小。理论上，信息内容不能仅通过表示的维度来评估，因为它丢弃了诸如相关结构的重要因素，而维度仅提供信息内容的粗略估计**。
2. **更高维度的表示在网络中更容易局部处理。在卷积网络中增加每个图块的激活允许更多解耦的特征，所产生的网络将训练更快**。
3. **空间聚合可以在较低维度嵌入上完成，而不会在表示能力上造成许多或任何损失**。例如，在执行更多展开（例如3×3）卷积之前，可以在空间聚合之前减小输入表示的维度，没有预期的严重不利影响。我们假设，如果在空间聚合上下文中使用输出，则相邻单元之间的强相关性会导致维度缩减期间的信息损失少得多。鉴于这些信号应该易于压缩，因此尺寸减小甚至会促进更快的学习。
4. 平衡网络的宽度和深度。**通过平衡每个阶段的滤波器数量和网络的深度可以达到网络的最佳性能，增加网络的宽度和深度可以有助于更高质量的网络。然而，如果两者并行增加，则可以达到恒定计算量的最佳改进**。因此，计算预算应该在网络的深度和宽度之间以平衡方式进行分配。

虽然这些原则可能是有意义的，但并不是开箱即用的直接使用它们来提高网络质量。我们的想法是仅在不明确的情况下才明智地使用它们。



## 3. Factorizing Convolutions with Large Filter Size

GoogLeNet网络[20]的大部分初始收益来源于大量地使用降维。这可以被视为以计算有效的方式分解卷积的特例。考虑例如1×1卷积层之后接一个3×3卷积层的情况。在视觉网络中，预期相近激活的输出是高度相关的。因此，我们可以预期，它们的激活可以在聚合之前被减少，并且这应该会导致类似的富有表现力的局部表示。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_1.png)

> 图1。
>
> Mini网络替换5×5卷积。

在这里，我们将在各种设定中探索卷积分解的其它方法，特别是为了提高解决方案的计算效率。由于Inception网络是全卷积的，每个权重对应每个激活的一次乘法。因此，任何计算成本的降低会导致参数数量减少。这意味着，通过适当的分解，我们可以得到更多的解耦参数，从而加快训练。此外，我们可以使用计算和内存节省来增加我们网络的滤波器组的大小，同时保持我们在单个计算机上训练每个模型副本的能力。



### 3.1. Factorization into smaller convolutions

具有较大空间滤波器（例如5×5或7×7）的卷积在计算方面往往不成比例地昂贵。例如，具有n个滤波器的5×5卷积在具有m个滤波器的网格上比具有相同数量的滤波器的3×3卷积的计算量高25/9=2.78倍。当然，5×5滤波器在更前面的层可以捕获更远的单元激活之间、信号之间的依赖关系，因此滤波器几何尺寸的减小带来了很大的表现力。然而，我们可以询问5×5卷积是否可以被具有相同输入尺寸和输出深度的参数较小的多层网络所取代。如果我们放大5×5卷积的计算图，我们看到每个输出看起来像一个小的完全连接的网络，在其输入上滑过5×5的块（见图1）。由于我们正在构建视觉网络，所以通过两层的卷积结构再次利用平移不变性来代替全连接的组件似乎是很自然的：第一层是3×3卷积，第二层是在第一层的3×3输出网格之上的一个全连接层（见图1）。通过在输入激活网格上滑动这个小网络，用两层3×3卷积来替换5×5卷积（比较图4和5）。

**该设定通过相邻块之间共享权重明显减少了参数数量**。为了分析预期的计算成本节省，我们将对典型的情况进行一些简单的假设：我们可以假设$n=αm$，也就是我们想通过常数$α$因子来改变激活/单元的数量。由于5×5卷积是聚合的，$α$通常比1略大（在GoogLeNet中大约是1.5）。用两个层替换5×5层，似乎可以通过两个步骤来实现扩展：在两个步骤中通过$\sqrtα$增加滤波器数量。为了简化我们的估计，通过选择$α=1$（无扩展），如果我们单纯地滑动网络而不重新使用相邻网格图块之间的计算，我们将增加计算成本。滑动该网络可以由两个3×3的卷积层表示，其重用相邻图块之间的激活。这样，我们最终得到一个计算量减少到$\frac{9+9}{25}×$的网络，通过这种分解导致了28％的相对增益。每个参数在每个单元的激活计算中只使用一次，所以参数计数具有完全相同的节约。不过，这个设置提出了两个一般性的问题：这种替换是否会导致任何表征力的丧失？如果我们的主要目标是对计算的线性部分进行分解，是不是建议在第一层保持线性激活？我们已经进行了几个控制实验（例如参见图2），并且在分解的所有阶段中使用线性激活总是逊于使用修正线性单元。我们将这个收益归因于网络可以学习的增强的空间变化，特别是如果我们对输出激活进行批标准化[7]。当对维度减小组件使用线性激活时，可以看到类似的效果。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_2.png)

> 图2。
>
> 两个Inception模型间几个控制实验中的一个，其中一个分解为线性层+ ReLU层，另一个使用两个ReLU层。在三亿八千六百万次运算后，在验证集上前者达到了`76.2% top-1`准确率，后者达到了`77.2% top-1`的准确率。



### 3.2. Spatial Factorization into Asymmetric Convolutions

上述结果表明，大于3×3的卷积滤波器可能不是通常有用的，因为它们总是可以简化为3×3卷积层序列。我们仍然可以问这个问题，是否应该把它们分解成更小的，例如2×2的卷积。然而，**通过使用非对称卷积，可以做出甚至比2×2更好的效果，即n×1。例如使用3×1卷积后接一个1×3卷积，相当于以与3×3卷积相同的感受野滑动两层网络**（参见图3）。如果输入和输出滤波器的数量相等，那么对于相同数量的输出滤波器，两层解决方案便宜33％。相比之下，将3×3卷积分解为两个2×2卷积表示仅节省了11％的计算量。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_3.png)

> 图3。
>
> 替换3×3卷积的Mini网络。网络的更低层由带有3个输出单元的3×1构成。

在理论上，我们可以进一步论证，可以通过1×n卷积和后面接一个n×1卷积替换任何n×n卷积，并且随着n增长，计算成本节省显著增加（见图6）。实际上，我们发现，采用这种分解在前面的层次上不能很好地工作，但是对于中等网格尺寸（在m×m特征图上，其中m范围在12到20之间），其给出了非常好的结果。在这个水平上，通过使用1×7卷积，然后是7×1卷积可以获得非常好的结果。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_4.png)

> 图4。
>
> [20]中描述的最初的Inception模块.

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_5.png)

> 图5。
>
> Inception模块中每个5×5卷积由两个3×3卷积替换，正如第2小节中原则3建议的那样。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_6.png)

> 图6。
>
> n×n卷积分解后的Inception模块。在我们提出的架构中，对17×17的网格我们选择n=7。（滤波器尺寸可以通过原则3选择）



## 4. Utility of Auxiliary Classifiers

（这一部分内容已过时，好像只在GAN中使用）

[20]引入了辅助分类器的概念，以改善非常深的网络的收敛。最初的动机是将有用的梯度推向较低层，使其立即有用，并通过抵抗非常深的网络中的消失梯度问题来提高训练过程中的收敛。Lee等人[11]也认为辅助分类器促进了更稳定的学习和更好的收敛。有趣的是，我们发现辅助分类器在训练早期并没有导致改善收敛：在两个模型达到高精度之前，有无侧边网络的训练进度看起来几乎相同。接近训练结束，辅助分支网络开始超越没有任何分支的网络的准确性，达到了更高的稳定水平。

另外，[20]在网络的不同阶段使用了两个侧分支。移除更下面的辅助分支对网络的最终质量没有任何不利影响。再加上前一段的观察结果，这意味着[20]最初的假设，这些分支有助于演变低级特征很可能是不适当的。相反，我们认为辅助分类器起着正则化项的作用。这是由于如果侧分支是批标准化的[7]或具有丢弃层，则网络的主分类器性能更好。这也为推测批标准化作为正则化项给出了一个弱支持证据。![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_7.png)

> 图7。
>
> 具有扩展的滤波器组输出的Inception模块。这种架构被用于最粗糙的（8×8）网格，以提升高维表示，如第2节原则2所建议的那样。我们仅在最粗的网格上使用了此解决方案，因为这是产生高维度的地方，稀疏表示是最重要的，因为与空间聚合相比，局部处理（1×11×1 卷积）的比率增加。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_8.png)

> 图8。
>
> 最后17×17层之上的辅助分类器。 侧头中的层的批标准化[7]导致`top-1 0.4％`的绝对收益。下轴显示执行的迭代次数，每个批次大小为32。



## 5. Efficient Grid Size Reduction

传统上，卷积网络使用一些池化操作来缩减特征图的网格大小。**为了避免表示瓶颈，在应用最大池化或平均池化之前，需要扩展网络滤波器的激活维度**。例如，开始有一个带有$k$个滤波器的$d×d$网格，如果我们想要达到一个带有$2k$个滤波器的$\frac{d}{2}×\frac{d}{2}$网格，我们首先需要用$2k$个滤波器计算步长为1的卷积，然后应用一个额外的池化步骤。这意味着总体计算成本由在较大的网格上使用$2d^2k^2$次运算的昂贵卷积支配。一种可能性是转换为带有卷积的池化，因此导致$2(\frac{d}{2})^2k^2$次运算，将计算成本降低为原来的四分之一。然而，由于表示的整体维度下降到$(\frac{d}{2})^2k$，会导致表示能力较弱的网络（参见图9），这会产生一个表示瓶颈。我们建议另一种变体，其甚至进一步降低了计算成本，同时消除了表示瓶颈（见图10）。我们可以使用两个平行的步长为2的块：$P$和$C$。$P$是一个池化层（平均池化或最大池化）的激活，两者都是步长为$2$，其滤波器组连接如图10所示。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_9.png)

> 图9。
>
> 减少网格尺寸的两种替代方式。左边的解决方案违反了第2节中不引入表示瓶颈的原则1。右边的版本计算量昂贵3倍。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Figure_10.png)

> 图10。
>
> 缩减网格尺寸的同时扩展滤波器组的Inception模块。它不仅廉价并且避免了原则1中提出的表示瓶颈。右侧的图表示相同的解决方案，但是从网格大小而不是运算的角度来看。



## 6. Inception-v2

在这里，我们连接上面的点，并提出了一个新的架构，在ILSVRC 2012分类基准数据集上提高了性能。我们的网络布局在表1中给出。注意，基于与3.1节中描述的同样想法，我们将传统的$7×7$卷积分解为3个$3×3$卷积。对于网络的Inception部分，我们在$35×35$处有33个传统的Inception模块，每个模块有$288$个滤波器。使用第5节中描述的网格缩减技术，这将缩减为$17×17$的网格，具有$768$个滤波器。这之后是图5所示的$5$个分解的Inception模块实例。使用图10所示的网格缩减技术，这被缩减为$8×8×1280$的网格。在最粗糙的$8×8$级别，我们有两个如图6所示的Inception模块，每个块连接的输出滤波器组的大小为2048。网络的详细结构，包括Inception模块内滤波器组的大小，在补充材料中给出，在提交的tar文件中的`model.txt`中给出。然而，我们已经观察到，只要遵守第2节的原则，对于各种变化网络的质量就相对稳定。虽然我们的网络深度是$42$层，但我们的计算成本仅比GoogLeNet高出约$2.5$倍，它仍比VGGNet要高效的多。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Table_1.png)

> 表1。
>
> 提出的网络架构的轮廓。每个模块的输出大小是下一模块的输入大小。我们正在使用图10所示的缩减技术的变种，以缩减应用时Inception块间的网格大小。我们用0填充标记了卷积，用于保持网格大小。这些Inception模块内部也使用0填充，不会减小网格大小。所有其它层不使用填充。选择各种滤波器组大小来观察第2节的原理4。



## 7. Model Regularization via Label Smoothing

我们提出了一种通过估计训练期间标签丢弃的边缘化效应来对分类器层进行正则化的机制。

对于每个训练样本$x$，我们的模型计算每个标签的概率$k∈{1…K}: p(k|x)=\frac{exp(zk)}{∑^K_{i=1}exp(z_i)}$。这里，$z_i$是对数单位或未归一化的对数概率。考虑这个训练样本在标签上的实际分布$q(k|x)$，因此归一化后$∑_kq(k|x)=1$。为了简洁，我们省略$p$和$q$对样本$x$的依赖。我们将样本损失定义为交叉熵：$ℓ=−∑^K_{k=1}log(p(k))q(k)$。最小化交叉熵等价于最大化标签对数似然期望，其中标签是根据它的实际分布$q(k)$选择的。交叉熵损失对于$z_k$是可微的，因此可以用来进行深度模型的梯度训练。其梯度有一个更简单的形式：$\frac{∂ℓ}{∂z_k}=p(k)−q(k)$，它的范围在$−1$到$1$之间。

考虑单个真实标签$y$的例子，对于所有$k≠y$，有$q(y)=1$，$q(k)=0$。在这种情况下，最小化交叉熵等价于最大化正确标签的对数似然。对于一个特定的样本$x$，其标签为$y$，对于$q(k)=δ_{k,y}$，最大化其对数概率，$δ_{k,y}$为狄拉克$δ$函数，当且仅当$k=y$时，$δ$函数值为1，否则为0。对于有限的$z_k$，不能取得最大值，但对于所有$k≠y$，如果$z_y≫z_k$——也就是说，如果对应实际标签的逻辑单元远大于其它的逻辑单元，那么对数概率会接近最大值。然而这可能会引起两个问题。首先，它可能导致过拟合：如果模型学习到对于每一个训练样本，分配所有概率到实际标签上，那么它不能保证泛化能力。第二，它鼓励最大的逻辑单元与所有其它逻辑单元之间的差距变大，与有界限的梯度$\frac{∂ℓ}{∂z_k}$相结合，这会降低模型的适应能力。直观上讲这会发生，因为模型变得对它的预测过于自信。

我们提出了一个鼓励模型不那么自信的机制。如果目标是最大化训练标签的对数似然，这可能不是想要的，但它确实使模型正规化并使其更具适应性。这个方法很简单。考虑标签$u(k)$的分布和平滑参数$ϵ$，*与训练样本$x$相互独立*。对于一个真实标签为$y$的训练样本，我们用

$$
q′(k|x)=(1−ϵ)δ_{k,y}+ϵu(k)
$$
代替标签分布$q(k|x)=δ_{k,y}$，其由最初的实际分布$q(k|x)$和固定分布$u(k)$混合得到，它们的权重分别为$1−ϵ$和$ϵ$。这可以看作获得标签$k$的分布如下：首先，将其设置为真实标签$k=y$；其次，用分布$u(k)$中的采样和概率$ϵ$替代$k$。我们建议使用标签上的先验分布作为$u(k)$。在我们的实验中，我们使用了均匀分布$u(k)=1/K$，以便使得
$$
q′(k)=(1−ϵ)δ_{k,y}+\frac{ϵ}{K}.
$$


我们将真实标签分布中的这种变化称为*标签平滑正则化*，或LSR。

注意，LSR实现了期望的目标，阻止了最大的逻辑单元变得比其它的逻辑单元更大。实际上，如果发生这种情况，则一个$q(k)$将接近$1$，而所有其它的将会接近$0$。这会导致$q′(k)$有一个大的交叉熵，因为不同于$q(k)=δ_{k,y}$，所有的$q′(k)$都有一个正的下界。

LSR的另一种解释可以通过考虑交叉熵来获得：

$$
H(q′,p)=−\sum_{k=1}^Klog \ p(k)q′(k)=(1−ϵ)H(q,p)+ϵH(u,p)
$$


因此，LSR等价于用一对这样的损失$H(q,p)$和$H(u,p)$来替换单个交叉熵损失$H(q,p)$。第二个损失惩罚预测的标签分布$p$与先验$u$之间的偏差，其中相对权重为$\frac{ϵ}{1−ϵ}$。注意，由于$H(u,p)=D_{KL}(u|p)+H(u)$和$H(u)$是固定的，因此这个偏差可以等价地被KL散度捕获。当$u$是均匀分布时，$H(u,p)$是度量预测分布p与均匀分布不同的程度，也可以通过负熵$−H(p)$来度量（但不等价）；我们还没有实验过这种方法。

在我们的$K=1000$类的ImageNet实验中，我们使用了$u(k)=1/10000$和$ϵ=0.1$。对于ILSVRC 2012，我们发现对于`top-1`错误率和`top-5`错误率，持续提高了大约$0.2%$（参见表3）。

> 标签平滑（Label smoothing），像L1、L2和dropout一样，是机器学习领域的一种正则化方法，通常用于分类问题，目的是防止模型在训练时过于自信地预测标签，改善泛化能力差的问题。
>
> NIPS 2019上的这篇论文[When Does Label Smoothing Help?](https://papers.nips.cc/paper/8717-when-does-label-smoothing-help.pdf)用实验说明了为什么Label smoothing可以work，指出标签平滑可以让分类之间的cluster更加紧凑，增加类间距离，减少类内距离，提高泛化性，同时还能提高Model Calibration（模型对于预测值的confidences和accuracies之间aligned的程度）。但是在模型蒸馏中使用Label smoothing会导致性能下降。



## 8. Training Methodology

我们在TensorFlow[1]分布式机器学习系统上使用随机梯度方法训练了我们的网络，使用了$50$个副本，每个副本在一个NVidia Kepler GPU上运行，批处理大小为$32$，$100$个epoch。我们之前的实验使用动量方法[19]，衰减值为$0.9$，而我们最好的模型是用RMSProp [21]实现的，衰减值为$0.9$，$ϵ=1.0$。我们使用$0.045$的学习率，每两个epoch以$0.94$的指数速率衰减。此外，阈值为$2.0$的梯度裁剪[14]被发现对于稳定训练是有用的。使用随时间计算的运行参数的平均值来执行模型评估。



## 9. Performance on Lower Resolution Input

视觉网络的典型用例是用于检测的后期分类，例如在Multibox [4]上下文中。这包括分析在某个上下文中包含单个对象的相对较小的图像块。任务是确定图像块的中心部分是否对应某个对象，如果是，则确定该对象的类别。这个挑战的是对象往往比较小，分辨率低。这就提出了如何正确处理低分辨率输入的问题。

普遍的看法是，使用更高分辨率感受野的模型倾向于导致显著改进的识别性能。然而，区分第一层感受野分辨率增加的效果和较大的模型容量、计算量的效果是很重要的。如果我们只是改变输入的分辨率而不进一步调整模型，那么我们最终将使用计算上更便宜的模型来解决更困难的任务。当然，由于减少了计算量，这些解决方案很自然就出来了。为了做出准确的评估，模型需要分析模糊的提示，以便能够“幻化”细节。这在计算上是昂贵的。因此问题依然存在：如果计算量保持不变，更高的输入分辨率会有多少帮助。确保不断努力的一个简单方法是在较低分辨率输入的情况下减少前两层的步长，或者简单地移除网络的第一个池化层。

为了这个目的我们进行了以下三个实验：

1. 步长为$2$，大小为$299×299$的感受野和最大池化。
2. 步长为$1$，大小为$151×151$的感受野和最大池化。
3. 步长为$1$，大小为$79×797$的感受野和第一层之后**没有**池化。

所有三个网络具有几乎相同的计算成本。虽然第三个网络稍微便宜一些，但是池化层的成本是无足轻重的（在总成本的1%1%以内）。在每种情况下，网络都进行了训练，直到收敛，并在ImageNet ILSVRC 2012分类基准数据集的验证集上衡量其质量。结果如表2所示。虽然分辨率较低的网络需要更长时间去训练，但最终结果却与较高分辨率网络的质量相当接近。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Table_2.png)

> 表2。
>
> 当感受野尺寸变化时，识别性能的比较，但计算代价是不变的。

但是，如果只是单纯地按照输入分辨率减少网络尺寸，那么网络的性能就会差得多。然而，这将是一个不公平的比较，因为我们将在比较困难的任务上比较一个便宜16倍的模型。

表2的这些结果也表明，有人可能会考虑在R-CNN [5]的上下文中对更小的对象使用专用的高成本低分辨率网络。



## 10. Experimental Results and Comparisons

表3显示了我们提出的体系结构（Inception-v2）识别性能的实验结果，架构如第6节所述。每个Inception-v2行显示了累积变化的结果，包括突出显示的新修改加上所有先前修改的结果。标签平滑是指在第7节中描述的方法。分解的$7×7$包括将第一个$7×7$卷积层分解成$3×3$卷积层序列的改变。BN-auxiliary是指辅助分类器的全连接层也批标准化的版本，而不仅仅是卷积。我们将表3最后一行的模型称为Inception-v3，并在多裁剪图像和组合设置中评估其性能。

我们所有的评估都在ILSVRC-2012验证集上的48238个非黑名单样本中完成，如[16]所示。我们也对所有50000个样本进行了评估，结果在`top-5`错误率中大约为$0.1%$，在`top-1`错误率中大约为$0.2%$。在本文即将出版的版本中，我们将在测试集上验证我们的组合结果，但是我们上一次对BN-Inception的春季测试[7]表明测试集和验证集错误趋于相关性很好。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Table_3.png)

> 表3。
>
> 单张裁剪图像的实验结果，比较各种影响因素的累积影响。我们将我们的数据与Ioffe等人[7]发布的单张裁剪图像的最好推断结果进行了比较。在“Inception-v2”行，变化是累积的并且接下来的每一行都包含除了前面的变化之外的新变化。最后一行是所有的变化，我们称为“Inception-v3”。遗憾的是，He等人[6]仅报告了10个裁剪图像的评估结果，但没有单张裁剪图像的结果，报告在下面的表4中。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Table_4.png)

> 表4。
>
> 单模型，多裁剪图像的实验结果，比较各种影响因素的累积影响。我们将我们的数据与ILSVRC 2012分类基准中发布的最佳单模型推断结果进行了比较。

![](/img/ClassicPaperTranslation/GoogleNet/V3/Table_5.png)

> 表5。
>
> 模型组合评估结果，比较多模型，多裁剪图像的报告结果。我们的数据与ILSVRC 2012分类基准数据集上发布的最好模型组合推断结果的比较。所有的结果，除了在验证集上的`top-5`模型组合结果。模型组合在验证集上取得了`3.46% top-5`错误率。



## 11. Conclusions

我们提供了几个设计原则来扩展卷积网络，并在Inception体系结构的背景下进行研究。这个指导可以导致高性能的视觉网络，与更简单、更单一的体系结构相比，它具有相对适中的计算成本。Inception-v3的最高质量版本在ILSVR 2012分类上的**单裁剪图像**评估中达到了$21.2\%$的`top-1`错误率和$5.6\%$的`top-5`错误率，达到了新的水平。与Ioffe等[7]中描述的网络相比，这是通过增加相对适中（$2.5times$）的计算成本来实现的。尽管如此，我们的解决方案所使用的计算量比基于更密集网络公布的最佳结果要少得多：我们的模型比He等[6]的结果更好——将`top-5(top-1)`的错误率相对分别减少了$25\%$ ($14\%$)，然而在计算代价上便宜了六倍，并且使用了至少减少了五倍的参数（估计值）。我们的四个Inception-v3模型的组合效果达到了$3.5％$，多裁剪图像评估达到了$3.5\%$的`top-5`的错误率，这相当于比最佳发布的结果减少了$25％$以上，几乎是ILSVRC 2014的冠军GoogLeNet组合错误率的一半。

我们还表明，可以通过感受野分辨率为$79×79$的感受野取得高质量的结果。这可能证明在检测相对较小物体的系统中是有用的。我们已经研究了在神经网络中如何分解卷积和积极降维可以导致计算成本相对较低的网络，同时保持高质量。较低的参数数量、额外的正则化、批标准化的辅助分类器和标签平滑的组合允许在相对适中大小的训练集上训练高质量的网络。
