# 1.背景
1.贝叶斯网络是一个DAG，学习DAG是一个比较艰巨的任务因为它的搜索空间与节点个数呈超指数关系，学习BN是一个NPhard的问题因此这方面有很多研究，  
2.基于打分函数的方法往往通过优化一个关于链接矩阵A以及观测样本的打分函数，在无环约束下寻找BN。  
3.目前有研究将无环约束转变为一个**连续优化问题**，利用的是矩阵指数
$$
A\omicron A
$$
得到这个连续优化的方法后，我们可以用一些黑盒的优化器对其进行求解。  
4.**忠实性假设**使得联合分布P中的条件独立性可以在有向无环图G中被反映出来，这是**从联合分布样本中恢复DAG的基础**——从一个未知联合分布中生成多个独立同分布的样本并用这些样本恢复该分布对应的DAG
# 2.相关工作
1.现存大量的DAG恢复方法，例如基于打分函数的方法、基于约束的方法，**传统的DAG寻找方法一般适用于离散的数据**  
2.zheng等人将离散搜索转化为了连续问题，使得可以使用神经网络等近似器  
3.









# 3.目标与工作
1.本文提出了一种利用**深度生成模型以及结构约束的变体**对DAG进行学习旨在更好地捕捉忠实于DAG的采样分布，这是由深度神经网络的显著成功所激发的，它可以说是通用的近似器。  
2.作者修改了基于打分函数方法的目标函数（一般的目标函数会对变量以及模型种类做出先验假设一但这个假设往往很难满足）  
3.作者利用变分推断以及用图神经网络优化一对自动编码器的参数  






# 4.模型贡献以及创新之处
1.模型基于VAE建立，连接矩阵可以看成是一个参数可以一起学习  
2.VAE可以处理连续或离散的数据  
3.图神经网络的使用使得模型可以处理标量与向量  
4.提出了一个更适合在当前深度学习平台下实施的非周期性约束的变体。




# 5.利用神经网络学习DAG结构
本文采用深度生成模型生成一个带权重的邻接矩阵，这个矩阵包含了SEM结构。设$X\in\mathbb{R}^{n\times b}$是由随机向量$X=(X_1.X_2,...,X_d)$的n个$i.i.d$观测值组成的数据矩阵，设$\mathbb{D}$表示$d$个节点上组成的DAGs  $G=<V,E>$的（离散）空间。给定X，我们寻求学习对应联合分布$\mathbb{P}(X)$的DAG （也称为贝叶斯网络）。
## 5.1线性结构方程模型
模型表达
$$
X=A^TX+Z \tag{1}
$$
本文中的变量定义有点不同。其中，当变量顺序按一定方式排列时，A常常可以表达为一个上三角矩阵，是m个变量的连接矩阵，其中$A\in \mathbb{R}^{m\times m}$，$X\in \mathbb{R}^{m\times d}$,X的每一行代表一个节点/变量，这里**变量本身可以是一条向量**，因此**每个样本都是二维的**，因为除了变量本身，还有变量的向量对应的维度，也就是一张表其实是一个样本，因为每个样本对应的每个变量/特征取值是一个向量。
$$
X=(I-A^T)^{-1}Z\tag{2}  
$$ 
从(2)式我们发现，对原始样本进行原始采样（ancestral sampling）相当于生成噪声Z($Z=(I-A^T)X$)。原始噪声的定义如下：  
>在有向图模型中采样，可以通过一个简单高效的过程从模型所表示的联合分布中产生样本，这个过程被称为**原始采样**。  
原始采样的基本思想是将图中的变量$X_i$使用拓扑排序，使得对于所有$i$和$j$，如果$X_i$是$X_j$的一个父节点，则$j$大于$i$。然后可以此顺序对变量进行采样。换句话说，我们可以首先采X_1~P(X_1)，然后采X_2~P(X_2|Pa(X_2))，以此类推，直到最后我们从$P(X_n|Pa(X_n))$中采样。只要不难从每个条件分布$X_i~P(X_i|Pa(X_i))$中采样，那么从整个模型中采样也是容易的,这样采样就和矩阵A是严格上三角矩阵是对应的。

传统的SEM方法无法表示变量之间的非线性关系，**因为这个框架中各个变量之间的联系使用的是邻接矩阵A，也即只能用线性叠加的方式表达各个变量之间的关系**，于是这篇文章利用神经网络（**图神经网络**）来对变量X进行处理，从而使其能够表达变量之间的非线性关系。

## 5.2图神经网络
我们可以将2式改写为
$$
X=f_A(Z)\tag{3}
$$
这个形式满足图神经网络的一般形式，以GCN为例
$$
X=\widehat{A}ReLU(\widehat{A}ZW^{1})W^2\tag{4}
$$
基于式（2）作者给出了一种新的图神经网络结构，类似于（2）的非线性扩展
$$
X=f_2((I-A^T)^{-1}f_1(Z))\tag{5}
$$
若5式中$f_2$可逆，可以得到
$$
f_2^{-1}(X)=f_1(Z)+A^Tf_2^{-1}(X)\tag{6}
$$
6是2的扩展版本，**后面作者将$f_1$设置为了恒等变换，$f_2$是一个MLP，于是相当于在原SEM基础上，增加了变量之间的非线性关系的表达。**
## 5.3VAE
### 5.3.1变分推断
变分推断问题可描述为
>假如我们有一些观测数据（Observations），这些样本的生成服从某个分布 P(X|Z)，可以看到，这个分布里X的生成还受到一些隐变量（latent variable）Z 的影响，但是我们并不知道Z的具体分布，我们只有观测到的样本X ，我们希望在给出X的情况下推出Z的分布，即P(Z|X)。

于是需要采用变分推断的手段恢复Z的分布，即表达为
![图 1](images/b8e23c335e50e440c741bac24fa7243b1a397355a257411e2be2738f7ff82bf1.png)  
最终这个式子可以转化为求解ELBO最大的问题
![图 2](images/71bdf173449ac64b0f86118b071353936b88f368c89dfccf631980ca8203adff.png)  
### 5.3.2VAE
VAE的示意图如下：
![图 2](images/dbc5e4609c1529b62d2d3328891d333317c5dadd3fbfd66e37d7f2ba6ba181e4.png) 
VAE的目标在于优化ELBO：
![图 1](images/dc08d1c271f6bc1efe9382424a72696e6cc1234039fa3d1f6bce0de15e8110ff.png)  
## 5.4模型框架以及损失项
### 5.4.1模型框架如下
![图 4](images/e91a5e2107246a7b7ac6526b2816925e971b9c464a5329aa13997ff218149015.png)  
### 5.4.2模型设置
模型中编码器：
$$
Z=f_4((I-A^T)f_3(X))\tag{7}
$$
解码器：
$$
X=f_2((I-A^T)^{-1}f_1(Z))\tag{8}
$$
其中$f_2$与$f_3$设置为多层感知器（MLP），而$f_4$与$f_1$设置成了恒等映射，所以**各个变量之间的非线性连接源自$f_1$与$f_3$的MLP**，具体可以表达为前的推导：
$$
Encoder:[M_Z|logS_Z]=(I-A_T)MLP(X,W^1,W^2)\tag{9}  
$$
$$
Decoder:[M_X|logS_X]=MLP((I-A^T)Z,W^3,W^4)\tag{10}
$$
## 5.5损失项构建
假设后验分布$q(Z|X)$以及似然分布$p(X|Z)$服从因子高斯分布，其实这里就是假设X与Z中所有变量本身是独立的了<font color=red>（**？？？存在问题**）</font>，即
$$
q(Z|x)=\prod_{i=0}^m\prod_{j=0}^dq(Z_{ij}|X)
$$
$$
p(X|Z)=\prod_{i=0}^m\prod_{j=0}^dp(X_{ij}|Z)
$$
ELBO的两部分分别计算如下：  
![图 1](images/790e3b0b7f73018cdb2d0a53d320193bbfaee8b8badea13503198646805c12ea.png)  

![图 2](images/49961798a9d3509c5aebd8d8a3451bd12ffefc3954ab6f3de6edd37cc820f1d8.png)  



## 5.6离散数据的处理
假如变量是离散时，即X是有限离散，每个变量有d个候选的取值。**于是作者在这里将样本中的每一行即每个变量设置为了独热向量。**
>独热编码（One-Hot Encoding），又称一位有效编码，其方法是使用N位状态寄存器来对N个状态进行编码，每个状态都有它独立的寄存器位，并且在任意时候，其中只有一位有效。即，只有一位是1，其余都是零值。

与独热向量对应，模型这里假设$p(X|Z)$服从因子分类分布,即
$$
p(X|Z)=\prod_{i=0}^m\prod_{j=0}^nP_{ij}^{(l)}
$$
于是在解码器部分使用了逐行的softmax：
$$
softmax(MLP((I-A^T)^{-1},Z,W^3,W^4))
$$
## 5.7无环约束的改进
### part1  DAG with NO TEARS的无环约束
在论文**DAGs with NO TEARS: Continuous optimization for structure learning**（以下简称NOTEAR）中将DAG学习的无环结构约束由传统的组合约束（离散）改进为了一种连续约束。具体如下所示
![图 3](images/e1536222589e17560f85788425797bd483dd5997793d434fb5a1279ae99f95c9.png)  
NOTEARS文章中提出使用函数$h(W)=0$来使得传统的约束连续化，$h(W)=0$一般需要一下特点：
![图 6](images/2291304bbb2dc5ecc654d426f3eb5a3931f6ba8aea7236810b3370fd837df09a.png)  
这篇文章中作者使用了，如下的$h(W)$:
$$
h(W)=tr(e^{W\omicron W})-d
$$
**这个函数设计的非常精妙**，其中$\omicron$表示把两个大小相同的矩阵对应元素相乘，这里是将为了$W\omicron W$非负。使用该方法的原因如下：（<font color=red>**这个函数的设计真的非常精妙**</font>）
![图 7](images/622a69bbadd9502fc8500923112dd9fb862a1c934e033c9e8e5afe889ed16bd2.png)  
### part2 本文中约束（也很精妙）并且考虑了不是所有平台都支持矩阵指数，所以提出了以下的模型
![图 8](images/9bfe03c19724e69bb865ab46aa39bdfbb32a4b0628c09c0ea81c3129e4bb6e38.png)  

## 5.8 模型训练 
那么经过刚才的转换之后，问题转化为了
$$
\begin{aligned}
min~~f(A,\theta)=-L_{ELBO} \\
s.t.~~ tr[(I+\alpha A\omicron A)^m]-d=0
\end{aligned}
$$
那么问题就转换为了求解一个带约束的最优解问题。在这个问题中需要优化的参数包括邻接矩阵A以及VAE中包含的参数$\theta~(\theta={W_1,W_2,W_3,W_4})$。作者在这里使用了增广拉格朗日方法（流程见下图，一般是在拉格朗日法的基础上加上惩罚函数法，然后循环固定参数以及变量来达到最优解）进行最优解解算。
>![图 1](images/56e69977894670eb0979a3a45642364d69cc10186df93771841290b81e712109.png)  
于是问题的增广拉格朗日式子可以表达为
$$
L_c(A,\theta,\lambda)=f(A,\theta)+\lambda h(A)+\frac{c}{2}\vert h(A)\vert^2
$$
假如$c=+\infty$，则最小化$L_c$需要$h(A)=0$，也就是满足无环约束。作者提出了一种求解最优解的方案：
$$
\begin{aligned}
step~1:~~~(A^k,\theta^k)=\underset{A^k,\theta^k}{argmin}L_{c^k}(A,\theta,\lambda^k) \\
step~2:~~~\lambda^{k+1}=\lambda^k+c^kh(A^k) \\
step~3:~~~c^{k+1}=\begin{cases}
    \eta c^k,~~\text{if}~\vert h(A^k)\vert>\gamma\vert h(A)^{k-1} \vert \\
    c^k,~~\text{otherwise}
\end{cases}
    
\end{aligned}

$$
step1一般就用一些黑盒的优化方法进行优化，比如梯度下降。
## 5.9与线性SEM的联系
该模型的损失函数由两部分构成:$D_{KL}(q(Z|X)\vert\vert p(Z))$以及$E_{q(Z|X)}[log~p(x|Z)]$构成，作者首先将VAE的变分部分剔除。于是，自编码器的损失函数为
$$
\frac{1}{2}\sum_{i=1}^m\sum_{j=1}^d(X_{ij}-\hat{X}_{ij})^2+\frac{1}{2}\sum_{i=1}^m\sum_{j=1}^dZ_{ij}^2
$$
变分部分剔除的话，也就代表着编码器部分的输出$M_Z=Z,S_Z=1$，解码器输出为$M_X=\hat{X},S_X=1$。于是原模型的损失项的两部分可以对应为上面的两项。而在NOTEARS一文中，也不包含神经网络部分，于是把神经网络部分去掉的话，编码器与解码器为
$$
\begin{aligned}
    Encoder:Z=(I-A^T)X\\
    Decoder:\hat{X}=(I-A^T)^{-1}Z
\end{aligned}
    
$$
很明显不存在重构误差，于是只剩下了正则化项：
$$
\frac{1}{2}\sum_{i=1}^m\sum_{j=1}^dZ_{ij}^2
$$
这就是NOTEARS一文的损失函数。