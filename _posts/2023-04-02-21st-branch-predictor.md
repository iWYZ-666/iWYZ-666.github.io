---
layout: post #Do not change.
category: [notes] #One, more categories or no at all.
title: "21世纪的分支预测技术" #Article title.
author:  iWYZ-666 #Author's nick.
fb_app_id: example
---


这篇文章主要探讨条件分支的预测。请先阅读90年代的分支预测技术一篇后再来阅读本文，以免前置知识缺失。

## perceptron大类

实际上perceptron在分支预测中被应用也是21世纪的事了，之前放在90年代那篇里面介绍是因为最早的perceptron的分支预测算法比较简单。但这篇论文的影响力还是相当大的，直接催生了一系列的分支预测算法。

### Fast Path-based Perceptron

在原始perceptron算法提出来后，虽然在原论文中讨论了perceptron方法的延迟和复杂度，但最终还是被人指出算法的延迟过大。Fast Path-based Perceptron使用history path作为记录，进行分支的预测。

![](/assets/images/fast_path-based_per_and_ori.png)

上图中左边的（a）是原perception算法，比如我要预测分支b<sub>t</sub>，那么我所用的权重都是通过分支b<sub>t</sub>找到的这一列（权重有8个），分支与分支之间的权重是没有关系的（独立的）。右边的（b）展示的path-based perceptron方法，预测b<sub>t</sub>的权重，所使用的的第一个权重是来自自己的x<sub>0</sub>，但第二个权重就是上一个分支的第二个权重x<sub>1</sub>，使用的第三个权重是上上个分支的第三个权重x<sub>2</sub>。**为什么这样做？**为的就是降低分支预测算法的延迟。我们看图（b），b<sub>t-7</sub>，它的权重x<sub>7</sub>会被分支b<sub>t</sub>使用，权重y<sub>6</sub>会被b<sub>t-1</sub>使用，权重z<sub>5</sub>会被b<sub>t-2</sub>使用。即一个分支被预测后，其第二个权重到第八个权重，就可以被计算用来预测将来的7个分支。所以说，path-based算法，把分支预测的运算提前了，等会看伪代码就明白了。

#### 伪代码

首先我们看预测部分的伪代码：

![](/assets/images/fast_path-based_predict.png)

我们进行一些说明。pc是分支指令的地址，i := pc mod n实际上应该写为hash(pc) mod n，实际算法必然使用某种哈希函数。而预测的算法，实际到第一个end if处就已经结束了（此时prediction已被赋值）。h是path的长度，比如上图中path中保存了前7个分支，h=7。W是权重矩阵。

SR是一个类似于队列的结构，在伪代码中每次都是取队尾（和平时的队列不同，取队头）。我们仍然结合（b）图来理解，假设现在我们预测$b_t$的方向，SR[h]中保存的是$x_1b_{t-1}+x_2b_{t-2}+x_3b{t-3}+...+x_7b_{t-7}$，其中b代表的是分支taken或者not taken（**需要注意的是，不是分支实际的taken与否，而是算法预测的taken与否**），taken就是1，not taken就是-1。SR[h]中的值是在预测$b_t$前就已经算好了，当预测$b_t$时，只需要y := SR[h] + x<sub>0</sub>即可。在原始的perceptron算法中，x<sub>0</sub>就是bias量，无需×别的东西。

我们再看for循环。我们说SR中的值是提前算好的，这里就是算SR中的值的过程。此时b<sub>t</sub>的第二个权重可以被用于预测下一个分支。我们将SR队尾去了，队头处push一个0。$SR[h] += W[i,1]\times prediction,SR[h-1]+=W[i,2]\times prediction,...,SR[1]+=W[i,h-1]$。这就是算SR的过程，此时的第二个权重 $\times$当前的预测值来预测下一个分支方向（SR[h]），第三个权重预测下下个分支方向（SR[h-1]），以此类推。 （注意，这里于伪代码有差异，但实际是一致的）

最终更新SG，SG存储的是分支预测的结果，SG := (SG << 1) or prediction是经典的式子了，不解释。

再看train部分的伪代码：

![](/assets/images/fast_path-based_train.png)

i，y<sub>out</sub>是上个伪代码中的i和y。prediction是预测的方向，outcome是实际的方向（taken or not taken）。我们知道W是权重矩阵，$W[i][j]$中i就是伪代码中的i，由分支指令的pc哈希而来，j代表的是第j + 1个权重（起始是0）。这里有个数组v，v[1]就是b<sub>t-1</sub>的pc哈希生成的值，v[2]就是b<sub>t-2</sub>的pc生成出来的值，以此类推。这里用v数组生成k，k用来定位W权重矩阵中哪组权重。SG := G是对SG的修正，SG保存的预测的方向，G保存的实际的方向，所以用G来修正SG。但SR := R这行伪代码极其不负责，这里的R代表的是修正后的SR，用它来修正SR，但怎么生成的，论文没说（看到图中右下角那个大大的not shown了么？）。最后介绍H数组，其保存前h个分支的预测方向。

个人认为$W[k,j]:=W[k,j]+...$这行代码挺关键的。我们说path-based可以降低延迟，不仅仅是权重用了别的分支的权重，我们知道算法是由$\sum W\times predicton$得到的，W是别的分支的W，prediction也是别的分支的prediction，那么只能在train的过程让这个权重和prediction为我这个分支所用。outcome和prediction（记住prediction是别的分支，outcome是自己这个分支）相同，+1，反之-1，感知机算法才成立。

#### R

很明显，这里的R非常难以修复，因为权重的个数是已知的，但有多少分支用了这些权重是未知的。论文故意对这块含糊其辞，但这里实际上是重中之重。R的状态保存是这个算法最复杂的地方。

#### 延迟

这篇论文分析了一些延迟方面的内容，因为暂时的研究方向与此无关（设计到一些电路设计，其实也没看懂），这块就不写了。

但论文中提到了一个降低延迟的技术，称为**overriding**。原理十分简单，就是用一种低延迟，精度低的分支预测算法作为第一层，高延迟，精度高的分支预测算法作为第二层。第二层与第一层结果不一致时就要修正（所以代价还挺大）。论文宣称fast path-based延迟是2个cycle，但在实验中还是使用了overriding技术。第一层算法是bimodal算法。

## O-GEHL

### 主体结构

![](/assets/images/gehl_predictor_figure.png)

GEHL分支预测器的核心思想非常简单，就是使用多张表，每张表中都是一些饱和计数器。每张表使用不同的Index方法获取饱和计数器的值。GEHL通过不同的Index方法从不同的表中取到计数器的值后，使用以下公式进行分支预测：
$$
S=\frac{M}{2}+\sum_{i=1}^MC(i-1)
$$

其中M是表的个数，C代表饱和计数器。当S > 0时，预测结果为taken，反之为not taken。这里的公式和感知机方法非常类似（所以O-GEHL也有感知机模型的缺点）。另外，论文没有解释为什么需要加上$\frac{M}{2}$。

饱和计数器的更新规则，与感知机方法一毛一样。预测失败或者成功但S < threshold时，对饱和计数器进行更新。需要预测倾向taken则 + 1，反之 - 1。

这里简单提一下Index，后文有详细介绍。GEHL方法，全称GEometric History Length，意思是指对于每张表的Index方法，所用到history length是不同的，且是指数增长的。对于第一张表，Table(0)，Index方法不依赖历史记录。我们假设对于Table(1)，其使用的历史记录长度为L(1)，则其它表Index方法使用的历史记录长度使用如下公式计算：
$$
L(i)=\alpha^{(i-1)}\times L(1)\ \ (i\textgreater 1)
$$
最后我们介绍一下CBP-1比赛中O-GEHL分支预测器所使用的参数。

8张表，从Table 0到Table 7。初始threshold设为8（和表的数量相同）。

### Tricks

O-GEHL的O，是优化（optimize）的意思。O-GEHL算法主要实现了2个优化tricks，一个是动态调整历史长度，另一个是动态调整threshold。

#### 动态调整历史长度

在CBP-1所给的trace benchmark中，有些倾向于使用短的历史记录来index（server benchmark，L(7)在50左右）。但有的workload更加偏向于使用较长的历史记录。

在O-GEHL中，使用Table 7的表现来改变某些表index时使用的历史长度。Table 7的entries加1个tag bit，在预测器更新时，遵循以下算法：

```text
if ((p != out) && (|S| <= threshold))
{
	if ((pc & 1) == Tag[indexT[7]])
		AC++;
	else
		Ac -= 4;
	if (AC == SaturatedPostive) UseLongHistories();
	if (AC ==  SaturatedNegative) UseShortHistories();
	Tag[indexT[7]] = (pc & 1);
}
// indexT[7]实际上是模拟代码中的表述，其实就是Table 7当前分支的index。
// Tag数组中存的是分支地址 & 1，代码最后一行进行更新
// 上述代码能区分index函数是不是hash到了别的分支上去了，是就 AC -= 4
```

上述代码的原理是使用一个9bit的饱和计数器，来衡量使用长历史记录的表的aliasing rate。Table 7是使用历史长度最长的表，如果其aliasing rate较低（AC比较大），则应该使用长历史记录，反之使用短历史记录。

如何调整使用的历史记录的长度？O-GEHL算法的设定是：Table 2的indx长度为L(2)或L(8)，Table 4的index长度为L(4)或L(9)，Table 6的index长度为L(6)或L(10)。

> CBP-1比赛在存储上是有限制的。O-GEHL的模拟代码中，并不是每个entry都对应一个tag bit，而是偶数index的entry才有对应的tag bit。也就是说只有有tag bit的entry更新时才会运行上面的代码，没有tag bit的entry就不运行上面的代码。注意O-GEHL是一个注重调参的算法，多少entry有tag bit和算法中的超参要匹配才能有好的效果。

#### 动态调整threshold

O-GEHL的论文宣称，在8张表的算法中，threshold选择5或者14，Branch MPKI只有0.5的区别（上文提到threshold初始设置为表的数量，即8）。虽然影响不大，但论文还是提出了一种动态调整threshold的方法。

论文认为，threshold与$\frac{NU_{miss}}{NU_{correct}}$有很强的关联。在某个threshold下，希望这个值尽量为1。

> $NU_{miss}$指因为预测错误导致的预测器更新的次数，$NU_{correct}$是指预测器正确但S未达到threshold而执行的更新。

论文使用一个7bit的饱和计数器TC，运行如下算法。

```text
if (p != out)
{
	TC++;
	if (TC == SaturatedPostive)
	{
		threshold++;
		TC = 0;
	}
}
else if (|S| < threshold)
{
	TC--;
	if (TC == SaturatedNegative)
	{
		threshold--;
		TC = 0;
	}
}
```

## PPM-Like

PPM-Like分支预测算法是Tage算法的前身（之一），Tage算法的论文省略了很多在PPM-Like论文中提到的东西，所以有必要介绍PPM-Like算法。

#### 主体结构

![](/assets/images/ppm-like.png)

上图是ppm-like分支预测器的主体结构。总共有5个bank，第一个bank使用分支pc地址的后12位寻址，总共有4K个entries，每个entry是1个3bit的饱和计数器和一个称为m的bit。后4个bank使用pc地址和分支历史记录xor哈希进行寻址，每个bank有1K个entries，每个entry有1个3bit的饱和计数器，8bit的tag位，和1bit的u。加在一起总共64KBits（CBP-1的比赛要求）。

### 预测策略

论文中称bank0（就是第一个bank）为bimodal table。需要预测时，预测器从5个bank中取出饱和计数器的值。bimodal table是一定能取出值的，但后4个bank由于tag bits的存在（这里的tag和Cache中的tag作用一致），不一定能取出对应的饱和计数器的值。

### 更新策略

#### 饱和计数器的更新

taken就+1，not taken就-1.

#### 预测错误时的更新

预测错误时，预测器希望在index函数使用更长历史记录的bank中分配一个entry（bank4错误了就没这待遇，因为它的index函数用了最长的分支历史记录）。预测器选取靠后的bank的饱和计数器的值为分支预测的结果。对于所有靠右（index使用更长分支历史）的bank，必然是tag miss的，我们取出这些bank对应entry的u bit，若u bit为1，我们不可以使用这个entry分配给当前分支，若u bit为0，则可以使用。
这里有2种情况：

1. 所有靠右bank对应的u bit都为1，我们就分配不了entry了，只能从这些靠右bank中随机选一个bank来分配；
2. 只要bank的u bit为0，就可以进行分配，在不止一张表中分配也没关系。比如bank1预测错误，需要分配，bank2-4对应entry的u bit都是0，都可以进行分配。

新entry的tag位需要修改为当前分支hash出来的tag（需要在该bank分配就说明之前必然tag miss）。u bit被置为0。饱和计数器的值由如下规则确定：

1. bimodal table中对应entry的m bit为1，则根据当前分支的方向而定，taken则置为4（weakly taken），not taken则置为3（weakly not taken）；
2. bimodal table中对应entry的m bit为0，则根据bimodal table对应entry的饱和计数器的值而定，若该值为taken，则置为4，反之为3。

#### 预测结果与bimodal table的预测结果不同时的更新

首先，出现不同就说明该分支在bank1-4中有tag hit。没有tag hit不可能不同。

由于结果只有taken和not taken，所以必然一方正确一方错误，就出现了2种情况。

1. 最终预测结果正确：bimodal table中对应entry的m bit和给出最终结果的bank的对应entry的u bit被置为1；
2. bimodal table正确：bimodal table中对应的entry的m bit和给出最终结果的bank的对应entry的u bit被置为0.

最后我们来看m bit和u bit的作用。u bit表示该entry useful，将其更新为1的时机，是其正确且bimodal table错误，说明该分支的预测需要历史记录的参与。且因为该entry有用，所有不希望被别的分支分配entry在这。m bit表示bimodal table中的entry不好，在分配新entry时不要参考，反之则需要参考。

> PPM-LIKE的论文有一半部分是在介绍自己的模型的，但那个模型没有半点指导意义。毕竟Tage算法吸收了PPM-LIKE后垄断该领域，但PPM-LIKE的作者并没有通过该模型优化PPM-LIKE算法。

### fold history

一种哈希方法，在CBP3的ISL-TAGE算法使用该方法作为哈希方法。

## Tage

Tage结合了GEHL（没有O）和PPM-LIKE算法的特点提出的分支预测算法。现代CPU的分支预测算法被认为基本都是基于Tage算法及其变种实现的。

### 主体结构

![](/assets/images/tage-overview.png)

上图就是Tage算法的主体结构（可以看出来和PPM-LIKE很像）。Tage算法主要改进的点在于index函数使用的历史长度基于GEHL，以及优化了算法的更新策略（更新策略占大头，因为PPM-LIKE算法index函数使用的历史长度也算是GEHL）。

### 预测策略

与PPM-LIKE算法一致，Tage算法更倾向于靠右的Table给出的预测结果。T0是使用分支PC来index的bimodal表，其饱和计数器为2bit。T1-T4的每个entry，由pred,tag和u组成。pred是3bit的计数器，tag bits作用还是tag的作用👻，u是1个2bit的计数器。

在预测时，预测器从T0-T4取出对应entry，T1-T4只有tag hit才可作为预测结果候选。比如T2和T4 tag hit了，根据靠右原则，最终预测结果使用T4给出的结果，我们称T4为provider，称T2为altpred。如果没有tag hit的table，我们称T0为altpred。

entry中的pred计数器给出预测结果，这是一个3bit的有符号计数器，其符号位指示预测结果。

### 更新策略

Tage算法的论文作者认为PPM-LIKE算法失败的原因在于更新策略过于简单。Tage算法主要改进了更新策略。

#### 更新useful counter u

当altpred提供的预测结果与provider提供的预测结果不一致时，必然有一方是错误的。u是无符号2bit计数器。

1. 当provider正确时，provider的u + 1；
2. 当altpred正确时，provider的u + 1。

每隔一段周期，u counters就会被reset。论文这里写的不是很明确，每过256K个branch，先将所有u counters的高位置为0，然后将u counters的低位置0。这里需要结合代码看。

#### 更新pred

taken就+1，not taken就-1，注意pred仍然是饱和的，不过是有符号的（感觉不过是为了和PPM-LIKE算法有差异，实际有无符号没啥区别）。

注意只更新provider的pred，altpred的pred是不更新的。

#### 预测错误时的更新

首先更新provider的pred counter。然后我们希望在靠右的table中给当前分支分配一个entry（只有一个table会被分配entry，PPM-LIKE会分配在多个bank中进行分配）。

> 在新一版的Tage算法论文中，作者提到在存储空间充足的情况下，可以考虑分配多个entry到不同table中。这和PPM-Like的做法是相同的。

会出现如下3种情况：

1. 只有一个靠右table的对应entry的u counter为0，则在这个table分配一个entry，对于那些对应entry的u counter不为0的table，所有的u counters都-1；
2. 若出现多个靠右table对应entry的u counter为0，则根据一定概率选择一个table来分配entry。举个例子，如果T<sub>i</sub>和T<sub>j</sub>对应的entry的u counter都为0（i < j），则T<sub>i</sub>应该被选中的概率是T<sub>j</sub>的2倍，也就是说算法更倾向于靠右table中靠左的table被分配entry；
3. 若没有靠右table对应entry的u counter为0，根据（1），u counters会-1，所以重复寻找必然能找到满足条件的table。

分配的新entry，其pred置为0（weakly taken），u置为0（strongly not useful）。👻，感觉u除了0以外含义类似啊，只是能多抗几轮减1😎。

### 对于更新策略的一些解释

u counter，u表示useful，其不为0则该entry不应该被分配新entry，这一点与PPM-LIKE算法是一致的。但是，PPM-LIKE缺少一些将u counter置为0的手段。比如一个entry，其上的分支实际已经很久被执行了，由于PPM-LIKE将u置为0的时机是最终预测结果与bimodal table的结果不一致，也就是说分支必须要被执行才有可能将u置为0，导致一些不可能再次执行的分支霸占了一些entries，导致空间上的浪费。周期性将u counters重置（每隔256K个分支），实际上是模拟LRU的时钟算法，可以将一些不执行的分支的u counter置为0，从而让其entry给别的分支让位。

> 在新一版的Tage算法论文中，useful counter改为1bit。算法维护一个8bit的计数器，在分配entry失败时-1，成功时+1，当计数器负饱和时，将u bit全部重置，这种方法的效果更好。

对于用不同概率选择table，论文称避免了ping-phenomenon，google了一下也不知道什么是ping-phenomenon。还有乒乓现象，并没有想通与这里的机制有什么关系。

### Tricks

我们可以看到，根据预测策略，Tage算法会使用tag hit的最靠右的table的结果作为预测结果，意味着算法倾向于index使用更长历史记录的table的结果。但这不总是最好的方式，在一些workload中，若provider中的entry是刚刚被分配的，这时使用altpred的结果作为预测结果效果更佳。论文说是使用一个全局的4bit counter来判断，但机制一字未提（要看代码）。

> 判断一个entry是否为刚刚分配的，使用正常方法会占用一定的空间。论文中认为u counter为0，pred为0或-1的entry是新分配的。

不同table的Tag Width也可以不同，如果Tage使用了十几个table，靠右的几个table可以使用更大的Tag W

### ITAGE

Tage算法也能作用在间接跳转的预测上。间接跳转不同于条件跳转只需要预测方向，间接跳转必须指定target address。下图展示了ITage预测器的整体结构

![](/assets/images/ITage.png)

pred被换成了target address，以及一个1bit的c。论文中称c是用来get some hysteresis on the predictor（我没理解这句话）。对于如何更新target address和c，论文并没有提，整个算法的主体框架与Tage是相同的。

### COTAGE

论文中提到，虽然ITage的精确度高于之前state-of-the-art的cascade间接分支预测器，但很多workload并没有很多间接跳转。ITage算法实现的复杂度远高于cascade。论文提出可以共用Tage和ITage的很多组件，比如tag匹配，这样可以降低硬件shi'xian

### 一些思考

从感知机算法开始，分支预测器就开始使用累加的方式判断最终的预测方向。但这导致过高的延迟，即使是path-based感知机也有2个cycle的延迟，同样基于累加的O-GEHL也会有这样的缺点。PPM-LIKE和Tage算法的改变就是仍然使用多张表，但不用累加的方式获得最终结果，而是使用index时使用历史长度最长的表

PPM-LIKE算法分配在预测错误时分配多个entry也是非常奇怪的做法，因为PPM-LIKE预测只会用到最靠右的bank和bimodal table，不会用到中间的bank，分配多个entry是很失败的做法。

## 参考资料

1. [Fast Path-based percetron论文](https://ieeexplore.ieee.org/document/1253199)
2. [PPM-Like](https://readpaper.com/pdf-annotate/note?pdfId=739087305792638976&noteId=1594407601154624256)
3. [Tage](https://readpaper.com/pdf-annotate/note?pdfId=4501428583444144129&noteId=733605343293190144)
4. [New Tage](https://readpaper.com/pdf-annotate/note?pdfId=4699917465522536449&noteId=1630713654485790464)
5. [O-GEHL](https://readpaper.com/pdf-annotate/note?pdfId=4699917537580679169&noteId=1560684841786524416)
