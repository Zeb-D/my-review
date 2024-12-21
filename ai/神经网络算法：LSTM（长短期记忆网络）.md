本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------



本文将从LSTM的本质、LSTM的原理、LSTM的应用三个方面，带您一文搞懂长短期记忆网络Long Short Term Memory | LSTM。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicklWBwUQOeVUL7faUpybAQficc4CmiaW61PtibAovQPhCcNaqulCxc51xvX9bkVH7FxSBP0ovuxfNy5Mw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



一、LSTM的本质

RNN 面临问题：RNN（递归神经网络）在处理长序列时面临的主要问题：短时记忆和梯度消失/梯度爆炸。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicklWBwUQOeVUL7faUpybAQficYvCPs5jPcMzcnNT0ALYLXkKGaUMgbawLyjWF7ib6UT8Qkov8ibbSk98A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



梯度更新规则

短时记忆问题

描述：RNN在处理长序列时，由于信息的传递是通过隐藏状态进行的，随着时间的推移，较早时间步的信息可能会在传递到后面的时间步时逐渐消失或被覆盖。

影响：这导致RNN难以捕捉和利用序列中的长期依赖关系，从而限制了其在处理复杂任务时的性能。

梯度消失/梯度爆炸问题

描述：在RNN的反向传播过程中，梯度会随着时间步的推移而逐渐消失（变得非常小）或爆炸（变得非常大）。

影响：梯度消失使得RNN在训练时难以学习到长期依赖关系，因为较早时间步的梯度信息在反向传播到初始层时几乎为零。梯度爆炸则可能导致训练过程不稳定，权重更新过大，甚至导致数值溢出。



LSTM解决问题：大脑和LSTM在处理信息时都选择性地保留重要信息，忽略不相关细节，并据此进行后续处理。这种机制使它们能够高效地处理和输出关键信息，解决了RNN（递归神经网络）在处理长序列时面临的问题。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficTldF1FssK1wTC5vSjx9kOebZgy0c6bFAibpNlCzoCAx9aaNPNZNOmcA/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

大脑记忆机制

大脑记忆机制：当浏览评论时，大脑倾向于记住重要的关键词。无关紧要的词汇和内容容易被忽略。回忆时，大脑提取并表达主要观点，忽略细节。

LSTM门控机制：LSTM通过输入门、遗忘门和输出门选择性地保留或忘记信息，使用保留的相关信息来进行预测，类似于大脑提取并表达主要观点。



二、LSTM的原理

RNN 工作原理：第一个词被转换成了机器可读的向量，然后 RNN 逐个处理向量序列。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQfic2A0ibKpticpsKZBFXR4pYdyQvxOe20a18DK6ibjxFxYjwBedXvfCueKmQ/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



逐一处理矢量序列

隐藏状态的传递

过程描述：在处理序列数据时，RNN将前一时间步的隐藏状态传递给下一个时间步。

作用：隐藏状态充当了神经网络的“记忆”，它包含了网络之前所见过的数据的相关信息。

重要性：这种传递机制使得RNN能够捕捉序列中的时序依赖关系。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQfic5JVbJ6hdOLw9zCpNs80fWBcD0zXPxzSrlFtD1gKhcEr1UiaibY2xmVfw/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



将隐藏状态传递给下一个时间步

隐藏状态的计算

细胞结构：RNN的一个细胞接收当前时间步的输入和前一时间步的隐藏状态。

组合方式：当前输入和先前隐藏状态被组合成一个向量，这个向量融合了当前和先前的信息。

激活函数：组合后的向量经过一个tanh激活函数的处理，输出新的隐藏状态。这个新的隐藏状态既包含了当前输入的信息，也包含了之前所有输入的历史信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficGogEydI5KqsEibpx5R0seQWUiaHibibsVsAjqfMeRUhKhKO5tmNRPgz3Dw/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



tanh激活函数（区间-1～1）

输出：新的隐藏状态被输出，并被传递给下一个时间步，继续参与序列的处理过程。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficQwsBibKIecP2s5Ce9nyuKFSxmv9Hs4xsY5THvLnSItkomSeq9Y3lPmg/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

LSTM工作原理![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/7TWRhh4xicklWBwUQOeVUL7faUpybAQficqNicYPiaFVSXt4OBibXicnp7AcvNvnXFZImQTf3wjkicY42XhahrXIeAv9g/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)













LSTM的细胞结构和运算

***\**\*\*\*\*\*\*\*\*\*LSTM的细胞结构和运算\*\*\*\*\*\*\*\*\*\**\***

- **输入门**

- - 作用：决定哪些新信息应该被添加到记忆单元中。
  - 组成：输入门由一个sigmoid激活函数和一个tanh激活函数组成。sigmoid函数决定哪些信息是重要的，而tanh函数则生成新的候选信息。
  - 运算：输入门的输出与候选信息相乘，得到的结果将在记忆单元更新时被考虑。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficDibxJj62qzibJWgqeibNrQpFJlBAafCT36CF1xu7E92XtKlWtk1q156zQ/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- ***\**\*\*\*\*\*输入门（sigmoid激活函数 + t\*\*\*\*\*\*\*\*\*\*anh激活函数\*\*\*\*\*\*\*\*\*\*）\*\*\*\*\*\**\***

- **遗忘门**

- - 作用：决定哪些旧信息应该从记忆单元中遗忘或移除。
  - 组成：遗忘门仅由一个sigmoid激活函数组成。

***\**\*\*\*\*\*![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficcvrDxKad39d8WgqQPibTUicrxibSliacZlEzL9pyIzcwYrXk40sSlj0h8Q/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)\*\*\*\*\*\**\***

***\**\*\*\*\*\*sigmoid\*\*\*\*\*\*\*\*\*\*激活函数\*\*\*\*\*\*\*\*\*\*（区间0～1）\*\*\*\*\*\**\***

- - 运算：sigmoid函数的输出直接与记忆单元的当前状态相乘，用于决定哪些信息应该被保留，哪些应该被遗忘。输出值越接近1的信息将被保留，而输出值越接近0的信息将被遗忘。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficgdUM2rxMKa2D0JwqicDVM1oWzf3u4Ll4NpYDYd7etKFBrvVCtiaficCuQ/640?wx_fmt=other&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- ***\**\*\*\*\*\*遗忘门（sigmoid激活函数）\*\*\*\*\*\**\***

- **输出门**

- - 作用：决定记忆单元中的哪些信息应该被输出到当前时间步的隐藏状态中。
  - 组成：输出门同样由一个sigmoid激活函数和一个tanh激活函数组成。sigmoid函数决定哪些信息应该被输出，而tanh函数则处理记忆单元的状态以准备输出。
  - 运算：sigmoid函数的输出与经过tanh函数处理的记忆单元状态相乘，得到的结果即为当前时间步的隐藏状态。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/7TWRhh4xicklWBwUQOeVUL7faUpybAQficia2gxMug3h7icdd2x4EDtHRw8aO7wLdMn5t0MSibCqTRIr1nKNw2K42jA/640?wx_fmt=gif&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- ***\**\*\*\*\*\*输出门（\*\*\*\*\*\*\*\*\*\*sigmoid激活函数 + \*\*\*\*\*\*\*\*\*\*tanh激活函数\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*）\*\*\*\*\*\**\***

***三、LSTM\**\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*的应用\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\**\****

### ***\**\*\*\*\*\*机器翻译：\*\*\*\*\*\**\***

***\**\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xickkibAru7pD2181gdguicgxTibUzf69nn0ocRgzmAJ2ngxaWpKLR5IZ4GHIq9PYq4USoUoZc2wkwZ4Jcg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\****

**应用描述：LSTM在机器翻译中用于将源语言句子自动翻译成目标语言句子。**

**关键组件**：

- **编码器（Encoder）**：一个LSTM网络，负责接收源语言句子并将其编码成一个固定长度的上下文向量。
- **解码器（Decoder）**：另一个LSTM网络，根据上下文向量生成目标语言的翻译句子。

**流程**：

1. **源语言输入**：将源语言句子分词并转换为词向量序列。
2. **编码**：使用编码器LSTM处理源语言词向量序列，输出上下文向量。
3. **初始化解码器**：将上下文向量作为解码器LSTM的初始隐藏状态。
4. **解码**：解码器LSTM逐步生成目标语言的词序列，直到生成完整的翻译句子。
5. **目标语言输出**：将解码器生成的词序列转换为目标语言句子。

**优化**：通过比较生成的翻译句子与真实目标句子，使用反向传播算法优化LSTM模型的参数，以提高翻译质量。



***\**\*\*\*\*\*\*\*情感分析：\*\*\*\*\*\*\*\*\****

***\**\*\*\*\*\*\*\*![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/7TWRhh4xicklWBwUQOeVUL7faUpybAQficrHbLX41AkLs2BDAxJPFHL9hQluY8GM4a573UHJB0miaFqEJSrLVMpSg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)\*\*\*\*\*\*\*\*\****

**应用描述：LSTM用于对文本进行情感分析，判断其情感倾向（积极、消极或中立）。**

**关键组件**：

- **LSTM网络**：接收文本序列并提取情感特征。
- **分类层**：根据LSTM提取的特征进行情感分类。

**流程**：

1. **文本预处理**：将文本分词、去除停用词等预处理操作。
2. **文本表示**：将预处理后的文本转换为词向量序列。
3. **特征提取**：使用LSTM网络处理词向量序列，提取文本中的情感特征。
4. **情感分类**：将LSTM提取的特征输入到分类层进行分类，得到情感倾向。
5. **输出**：输出文本的情感倾向（积极、消极或中立）。

**优化**：通过比较预测的情感倾向与真实标签，使用反向传播算法优化LSTM模型的参数，以提高情感分析的准确性。







https://mp.weixin.qq.com/s/p1jmj__DQIwDMDTCExHs5Q