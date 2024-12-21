本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xickmMj3AtII9EUZM4twd372iaZEqxoaB6NeAgvGUA0jS7QqtloP5DJ9KZwbPeUXDh8rbERXglqV5bbLg/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)



一、RNN编码器-解码器架构

序列到序列模型（Seq2Seq）：Seq2Seq模型的目标是将一个输入序列转换成另一个输出序列，这在多种应用中都具有广泛的实用价值，例如语言建模、机器翻译、对话生成等。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xickkUdiarScBn7L7yKAY3LnQ3FATDUrOSHXicBqVccLJvZAxFX2xia5EHkLia5UicuOrxUeWdVbQZXpQqGlQ/640?wx_fmt=gif&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\*\**\***













RNN编码器-解码器架构：Transformer出来之前，主流的序列转换模型都基于复杂的循环神经网络（RNN），包含编码器和解码器两部分。当时表现最好的模型还通过注意力机制将编码器和解码器连接起来。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickk4J3S5USf73apiaQEPWCo7PIe8g0UTXnicJ2kUAYMyA3keZNuvHNVQicjHTgWeV6WqHEILib5JTU6G9Q/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\****













在Seq2Seq框架中，RNN的作用主要体现在两个方面：编码和解码。

编码器：RNN接收输入序列，并逐个处理序列中的元素（如单词、字符或时间步），同时更新其内部状态以捕获序列中的依赖关系和上下文信息。这种内部状态，通常被称为“隐藏状态”，能够存储并传递关于输入序列的重要信息。

**![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicknaJQWwbKG5eBBUiaab7ibNu1QXsXOBZicsBWreL7ywMHZ3ibpUumB8ET7LibOGmCx9J7XibEKEyKP6FY1w/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)**











RNN编码器-解码器架构



解码器：RNN使用编码阶段生成的最终隐藏状态（或整个隐藏状态序列）作为初始条件，生成输出序列。在每一步中，解码器RNN都会根据当前状态、已生成的输出和可能的下一个元素候选来预测下一个元素。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicknF0tTdLFSQd2k6sUQnBIuCyFYRwf1xFynoC2MIf8njZgzBwIjoruyxPianPVDIO79wxIIcGc8sGtQ/640?wx_fmt=gif&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

因此，循环神经网络（RNN）、特别是长短时记忆网络（LSTM）和门控循环单元网络（GRU），已经在序列建模和转换问题中牢固确立了其作为最先进方法的地位。

**![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xickmgNoQcOziarUJAUGPQVZ3ibEtYic7KksW2UgsKNGdnX7TBicvQAdsPzXIuB1hhPq5K1V6p98MlcNcOsA/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)**









RNN LSTM GRU



[神经网络算法 - 一文搞懂RNN（循环神经网络）](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247485311&idx=1&sn=e0c1dc8d9035aabd1a3472436263f3fd&chksm=c26ee560f5196c76e4f509acff6c398a40077c0104d6fec4f2c93d50849f94ce68dc8a243062&scene=21#wechat_redirect)

[神经网络算法 - 一文搞懂LSTM（长短期记忆网络）](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247485606&idx=1&sn=8c70d3263d3749d5c06a676734c4164c&chksm=c26eeab9f51963af44abf80bd6e60192a597e4e1112e9c274d116e03300fa869b35bfb3c45dd&scene=21#wechat_redirect)

同时，RNN存在一个显著的缺陷：处理长序列时，会存在信息丢失。

编码器在转化序列x1, x2, x3, x4为单个向量c时，信息会丢失。因为所有信息被压缩到这一个向量中，处理长序列时，信息必然会丢失。![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickmNb66CPK1ibatV7jict9p6dHkGRA1V3vvgnRmMgHGvxfZvQG2plz089wuF7YVOKwZIrLahbdu0xdBQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)\*\*\*\*\*\**\***













RNN编码器-解码器架构





二、Transformer

Transformer起源：Google Brain 翻译团队通过论文《Attention is all you need》提出了一种全新的简单网络架构——Transformer，它完全基于注意力机制，摒弃了循环和卷积操作。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xickmMj3AtII9EUZM4twd372iaZ4oxybaHsx6P1dm7kibC1r8wFz59xw4uyZsyQXKOYYQPMd7l2RiaWCUAw/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

注意力机制是全部所需

注意力机制：一种允许模型在处理信息时专注于关键部分，忽略不相关信息，从而提高处理效率和准确性的机制。它模仿了人类视觉处理信息时选择性关注的特点。![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickmMj3AtII9EUZM4twd372iaZA00P1YWjnoD0ib712U0WFqnIBW0G8sAfjP8Oq4GHzB3VaVL5hAP7ciaw/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\*\*\*\*\****













注意力机制

当人类的视觉机制识别一个场景时，通常不会全面扫描整个场景，而是根据兴趣或需求集中关注特定的部分，如在这张图中，我们首先会注意到动物的脸部，正如注意力图所示，颜色更深的区域通常是我们最先注意到的部分，从而初步判断这可能是一只狼。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickmMj3AtII9EUZM4twd372iaZAicu1wCJt5giaFkbEt2XjFMGmHNvvamtmicUhL88YAcRky9zhice1sNQZw/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

注意力机制

[Transformer动画讲解 - 注意力机制](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247488859&idx=1&sn=60edb5a51d2d328b39f38a7b3be96926&chksm=c26ef744f5197e5288516187a29784f31404de80ea0429b9b90509919300e9bc192c42285d92&scene=21#wechat_redirect)***\**\*\*\*
\*\*\*\*\****

注意力计算Q、K、V：*****\**\*\*\*注意力机制通过查询（Q）匹配键（K）计算注意力分数（向量点乘并调整），将分数转换为权重后加权值（V）矩阵，得到最终注意力向量。*

注意力分数是量化注意力机制中信息被关注的程度，反映了信息在注意力机制中的重要性。



![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickkrnwr49fbMXMkia8ahkaJTQ2CqKmve9WT8hqyAGH7Oziczfy6qu4QKJA0HoiaM8GwcH3CI5ZeicWxMLQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\*\*\*\*\****











Q、K、V计算注意力分数

[Transformer动画讲解 - 注意力计算Q、K、V](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247489020&idx=1&sn=15d9be0a8ad752cb74a5c86c37bcdea6&chksm=c26ef7e3f5197ef5c2cd6662aa9fe662df44192452d560aaf7f5ab7dde6f34377cfae4f777da&scene=21#wechat_redirect)

Transformer本质：Transformer是一种基于自注意力机制的深度学习模型，为了解决RNN无法处理长序列依赖问题而设计的。![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickmh2icco34tMUkdeuNV6LxbInGwfQDhMTfrcbAspEsTUSjGVMqicLicZPKkvFjIxYagicwUicxGmHTl0dw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)













Transformer vs RNN

Transformer总体架构：Transformer也遵循编码器-解码器总体架构，使用堆叠的自注意力机制和逐位置的全连接层，分别用于编码器和解码器，如图中的左半部分和右半部分所示。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xickk4J3S5USf73apiaQEPWCo7PkOcibhCoz6u9Jib7tLHW6WUaaiaibly9s1eumlsONEObHdg03yVzicsBM0A/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\*\*\*\*\*\*\*\*\*\**\***























Transformer的架构

- Encoder编码器：Transformer的编码器由6个相同的层组成，每个层包括两个子层：一个多头自注意力层和一个逐位置的前馈神经网络。在每个子层之后，都会使用残差连接和层归一化操作，这些操作统称为Add&Normalize。这样的结构帮助编码器捕获输入序列中所有位置的依赖关系。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicklUgIiaibicDhZdYceN6GnPwMHKQFyLZFmibsDJCdBfY1s8Kn3rK1l3y8Fhn3uFPL6nnBp92805jQllLg/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

Encoder（编码器）架构

- Decoder解码器：Transformer的解码器由6个相同的层组成，每层包含三个子层：掩蔽自注意力层、Encoder-Decoder注意力层和逐位置的前馈神经网络。每个子层后都有残差连接和层归一化操作，简称Add&Normalize。这样的结构确保解码器在生成序列时，能够考虑到之前的输出，并避免未来信息的影响。

***\**\*![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicklUgIiaibicDhZdYceN6GnPwMHWEUiav6GOh1wB3AAiaqUR52nKtwOUjHxPjSKUy0g6M05oJB1DKXfw6Lg/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\*\**\***















Decoder（解码器）架构



[神经网络算法 - 一文搞懂FFNN（前馈神经网络）](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247487986&idx=1&sn=b0a5ee3a258672c679c50c8799ec56b0&chksm=c26ef3edf5197afb08294882bba88ad6b059266a7d9684b174b912c88a7a1fdc2beec11f0c01&scene=21#wechat_redirect)\*\**\***

Transformer核心组件：Transformer模型包含输入嵌入、位置编码、多头注意力、残差连接和层归一化、带掩码的多头注意力以及前馈网络等组件。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xickmgNoQcOziarUJAUGPQVZ3ibEibiaiaUiateMVWSok00AfEKhletTNia3icXC89UEWKdAmJvN998426Q3Cm1Q/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)\*\**\***





Transformer的核心组件

输入嵌入：将输入的文本转换为向量，便于模型处理。

位置编码：给输入向量添加位置信息，因为Transformer并行处理数据而不依赖顺序。

多头注意力：让模型同时关注输入序列的不同部分，捕获复杂的依赖关系。

残差连接与层归一化：通过添加跨层连接和标准化输出，帮助模型更好地训练，防止梯度问题。

带掩码的多头注意力：在生成文本时，确保模型只依赖已知的信息，而不是未来的内容。

前馈网络：对输入进行非线性变换，提取更高级别的特征。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicklUgIiaibicDhZdYceN6GnPwMHHPic5TdzNibupYdCPIy4a1cyibzuR3epHSsPxHouNE2KXW6bH5bCHXpiaQ/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

Transformer的核心组件

Transformer数据流转：可以概括为四个阶段，Embedding（嵌入）、Attention（注意力机制）、MLPs（多层感知机）和Unembedding（从模型表示到最终输出）。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/9FAARY9pSvibVB4tOAjF4BYonGs3JDUuaicut3O5nmA7FX29JGTu7918DdBkrcy3ibJCpn2RF8nutpLwQJ39vSgWA/640?wx_fmt=gif&from=appmsg&wxfrom=5&wx_lazy=1&tp=wxpic)

Embedding -> Attention -> MLPs -> Unembedding

[Transformer动画讲解 - 数据处理的四个阶段](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247488783&idx=1&sn=8731571cfbc950448fdc9d8eb3f7f5fc&chksm=c26ef710f5197e064a19aa5dec3c519e033192e754e45a8d1909221fb5bf718d68bdf22d5ef8&scene=21#wechat_redirect)

Transformer注意力层：

在Transformer架构中，有3种不同的注意力层（Self Attention自注意力、Cross Attention 交叉注意力、Causal Attention因果注意力）编码器中的自注意力层（Self Attention layer）：

编码器输入序列通过Multi-Head Self Attention（多头自注意力）计算注意力权重。

解码器中的交叉注意力层（Cross Attention layer）：编码器-解码器两个序列通过Multi-Head Cross Attention（多头交叉注意力）进行注意力转移。

解码器中的因果自注意力层（Causal Attention layer）：解码器的单个序列通过Multi-Head Causal Self Attention（多头因果自注意力）进行注意力计算

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xickkZu1PvjLFQBPTyvahvk9m7jo6fIj9ER3Lx6Ns5A7VgLicF3bqBsxpYY6p91loic1p7T4ms2icEA8bdA/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

Transformer注意力层

[神经网络算法 - 一文搞懂Transformer中的三种注意力机制](http://mp.weixin.qq.com/s?__biz=MzkzMTEzMzI5Ng==&mid=2247487955&idx=1&sn=54e5f2b6642d77252a5b36577ed36d66&chksm=c26ef3ccf5197ada6f052ea28194859a8213ed87b579cc8db959631bfa4de0fdc17a5e3c9e91&scene=21#wechat_redirect)



https://mp.weixin.qq.com/s/DenAOo2flPF3S9NSB4qe-Q
