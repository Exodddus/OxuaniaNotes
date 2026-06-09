# 注意力机制

## 不随意线索与Q, K, V
人类的注意力会被两种事物吸引：随意线索，不随意线索。其中不随意线索是不由自主注意到的，如颜色鲜艳的杯子；随意线索是经大脑控制有意识地集中注意力观察到的事务，如有意识的想要读某一本书。

卷积、全连接、池化层都只考虑不随意线索，而注意力机制则显式地考虑随意线索。所以线索被称为查询（query），每个输入是一个不随意线索（key）和值（value）组成的键值对，我们通过注意力池化层来有偏向性地选择某些输入。

可一般地写作 $f(x) = \Sigma_{i} \alpha (x, x_i) y_i$，其中 $\alpha(x, x_i)$ 是注意力权重。

## 非参注意力池化层


- kernel regression
![[Pasted image 20260417165741.png]]

$K(x)$是一个函数，用于衡量$x$和$x_i$之间的距离
高斯核 $K(u) = \frac{1}{\sqrt{2\pi}} \exp\left(-\frac{u^2}{2}\right)$

## 参数化注意力机制

在此基础上，引入可学习的变量 $w$，得到 $f(x) = \sum_{i=1}^{n} \mathrm{softmax}\left( -\frac{1}{2} \left( (x - x_i) w \right)^2 \right) y_i$

## 注意力分数


可学参数：$\mathbf{W}_k \in \mathbb{R}^{h \times k}, \mathbf{W}_q \in \mathbb{R}^{h \times q}, \mathbf{v} \in \mathbb{R}^{h}$ $$a(\mathbf{k}, \mathbf{q}) = \mathbf{v}^T \tanh(\mathbf{W}_k \mathbf{k} + \mathbf{W}_q \mathbf{q}) $$等价于将key和value合并起来后放入到一个隐藏大小为 $h$ 输出大小为 1 的单隐藏层MLP。

如果 query 和 key 都是同样的长度 $\mathbf{q}, \mathbf{k}_i \in \mathbb{R}^d$，那么可以 $$a(\mathbf{q}, \mathbf{k}_i) = \langle \mathbf{q}, \mathbf{k}_i \rangle / \sqrt{d}$$ 向量化版本：

- $\mathbf{Q} \in \mathbb{R}^{n \times d}, \mathbf{K} \in \mathbb{R}^{m \times d}, \mathbf{V} \in \mathbb{R}^{m \times v}$ 
- 注意力分数：$a(\mathbf{Q}, \mathbf{K}) = \mathbf{Q}\mathbf{K}^T / \sqrt{d} \in \mathbb{R}^{n \times m}$
- 注意力池化：$f = \mathrm{softmax}\left(a(\mathbf{Q}, \mathbf{K})\right) \mathbf{V} \in \mathbb{R}^{n \times v}$

注意力分数是query和key的相似度，未经normalize；注意力权重是分数的softmax结果。
常见的分数计算方法是：
1. 将 q 和 k 合并，进入单输出单隐藏层的 MLP
2. 将 q 和 k 做内积

## seq2seq with attention

机器翻译时，每个生成的次可能相关于原句子中不同的词。
编码器对每个词的输出作为 key 和 value，在这里这两个值是相同的。解码器 RNN 对上一个词的输出是 query。注意力的输出和下一个词的词嵌入合并进入 RNN 输入。
第一次 q 来自于结束符。


# 自注意力

将输入 $\mathbf{x}_i$ 本身同时作为 q, k, v
自注意力可以计算序列中任意两个位置的关系，在序列较长时计算量很大，但是完全并行，且从任意输入到任意输出只有一层计算，很适合在 GPU / TPU 上进行训练。

![[Pasted image 20260422164728.png]]

# 位置编码

跟CNN、RNN不同，自注意力本身没有记录位置信息，一个解决这个问题的办法是加入位置编码。
它不改变注意力机制本身，而是把位置信息注入到输入里。
- 假设长度为 $n$ 的序列是 $\mathbf{X} \in \mathbb{R}^{n \times d}$，那么使用位置编码矩阵 $\mathbf{P} \in \mathbb{R}^{n \times d}$ 来输出 $\mathbf{X} + \mathbf{P}$ 作为自编码输入 
- $\mathbf{P}$ 的元素如下计算： $$ p_{i,2j} = \sin\left(\frac{i}{10000^{2j/d}}\right), \quad p_{i,2j+1} = \cos\left(\frac{i}{10000^{2j/d}}\right) $$
偶数列与该偶数+1列是同参数的sin cos函数，因此是平移的关系。每个样本不同的特征加入的值是不一样的，不同样本加入的值也是不一样的。

![[Pasted image 20260422165444.png]]

列数较小的波动比较密集，列数较大的波动比较缓慢

![[Pasted image 20260422165756.png]]

用 sin cos 可以实现相对位置信息的编码，位于 $i+\delta$ 处的位置编码可以通过位置 $i$ 处的位置编码现行投影来表示。
BERT 用的是可学习的位置编码方法。

# Transformer

基于编码器-解码器架构来处理序列对，纯基于注意力。

![[Pasted image 20260422171015.png]]

原信息经过 Embedding ，再加入位置信息，然后输入多头注意力机制。Pisitionwise FFN 就是一个全连接层。
## 多头注意力
对同一 key, value, query，希望抽取不同的信息，例如短距离关系和长距离关系。
![[Pasted image 20260422211235.png]]
- query $\mathbf{q} \in \mathbb{R}^{d_q}$，key $\mathbf{k} \in \mathbb{R}^{d_k}$，value $\mathbf{v} \in \mathbb{R}^{d_v}$ 
- 头 $i$ 的可学习参数 $\mathbf{W}_i^{(q)} \in \mathbb{R}^{p_q \times d_q}, \mathbf{W}_i^{(k)} \in \mathbb{R}^{p_k \times d_k}, \mathbf{W}_i^{(v)} \in \mathbb{R}^{p_v \times d_v}$ 
- 头 $i$ 的输出 $\mathbf{h}_i = f\left(\mathbf{W}_i^{(q)} \mathbf{q}, \mathbf{W}_i^{(k)} \mathbf{k}, \mathbf{W}_i^{(v)} \mathbf{v}\right) \in \mathbb{R}^{p_v}$ 
- 输出的可学习参数 $\mathbf{W}_o \in \mathbb{R}^{p_o \times h p_v}$ 
- 多头注意力的输出 $$ \mathbf{W}_o \begin{bmatrix} \mathbf{h}_1 \\ \vdots \\ \mathbf{h}_h \end{bmatrix} \in \mathbb{R}^{p_o} $$
## 有掩码的多头注意力

解码器对序列中一个元素输出时，不考虑该元素之后的元素。可以通过掩码来实现。也就是在计算 $\mathbf{x}_i$ 输出时，加装当前序列长度为 $i$。

## 基于位置的前馈网络 Positionwise FFN
将输入形状由 $(b, n, d)$ 变换成 $(bn, d)$，输出形状由$(bn, d)$转变为 $(b, n, d)$，作用类似两个全连接层。

## 层归一化 layer normalisation

## 信息传递

编码器和解码器之间传递的是编码器中的输出 $\mathbf{y}_1, \dots, \mathbf{y}_n$，将其作为解码器中第 $i$
个Transformer 块中多头注意力的 key 和 value，而它的 query 来自目标序列。编码器和解码器中快的个数和输出维度都是一样的。

预测第 t+1 个输出时，解码器中输入前 t 个预测值，前 t 个预测值作为 key 和 value，第 t 个预测值还作为 query

