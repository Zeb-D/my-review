本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

复习了之前看的《机器学习》关于特征选择与稀疏学习这章节，就用了向量和词频这块内容。

词向量技术如同一位“语言炼金术师”，它把“我”、“北京”、“开心”这些符号，熔炼成带有密码的数字向量。从最原始的One-Hot编码（像给每个词发唯一身份证）到TF-IDF（像给词语贴重要性标签），再到Word2Vec、GloVe等模型带来的语义深度捕捉，最后是现在最先进的编码器和解码器模型，这些技术已经极大地推动了自然语言处理的发展。

尽管技术越来越新，但是业务中总会有各种各样的问题，遇到这种情景你该怎么办呢？

当老板要求明天上线一个文本分类系统时，你是花3天调BERT模型，还是用2小时写TF-IDF+XGBoost？

明明ChatGPT已经能写诗，为什么电商大厂还在用20年前的技术检测刷单评论？

本文将撕开算法黑箱，带你亲历NLP史上最隐秘的生存法则——用80分的算法解决90分的业务问题，才是工程师的顶级智慧。



### One-Hot编码

由于计算机只能读懂二进制，所以直接喂给计算机自然语言计算机是不能工作的。

一个很简单的思路就是用0和1等二进制信息来表示自然语言，于是便有了one-hot编码表示自然语言的形式。

举个例子，假如全世界的词突然减少到了只剩下7个词，即["我", "要", "去", "北京", "想想", "就", "开心"]，那么全世界的词表也就是["我", "要", "去", "北京", "想想", "就", "开心"]，

那么每个词的one-hot编码为：

"我" → [1, 0, 0, 0, 0, 0, 0]

"要" → [0, 1, 0, 0, 0, 0, 0]

"北京" → [0, 0, 0, 1, 0, 0, 0]

“我要去北京”  → [1, 1, 1, 1, 0, 0, 0]

代码示例：

```
# Press the green button in the gutter to run the script.
if __name__ == '__main__':
    # 构建数组
    print("onehot")
    from sklearn.preprocessing import OneHotEncoder

    encoder = OneHotEncoder()
    vectors = encoder.fit_transform([["我"], ["要"], ["去"], ["北京"], ["想想"], ["就"], ["开心"]])
    print("vectors:", vectors)

```

输出：

```
onehot
vectors:   (np.int32(0), np.int32(5))	1.0
  (np.int32(1), np.int32(6))	1.0
  (np.int32(2), np.int32(1))	1.0
  (np.int32(3), np.int32(0))	1.0
  (np.int32(4), np.int32(4))	1.0
  (np.int32(5), np.int32(2))	1.0
  (np.int32(6), np.int32(3))	1.0
```



核心优势解析

数学可解释性：每个词对应唯一正交向量，保证词间独立性

零计算成本：无需训练即可生成特征表示



缺陷

维度灾难：词表规模（N）决定向量维度（N维），若词汇量达3万，每个词需3万维稀疏向量，存储和计算效率极低。

语义鸿沟：Onehot编码无法捕捉词汇之间的语义关系，例如“猫”和“狗”在语义上更为接近，但通过onehot编码后，从向量角度来看，不同的词向量是正交的，所以相似度为零。

特征稀疏：实际文本中99.9%的维度值为0





### 词袋模型（Bag of Words）

有了one-hot的基础之后，就引出词袋模型了。

1.1 核心思想
词袋模型在One-Hot的"存在性判断"基础上，引入词频统计这一重要性维度。其核心假设是：词语在文档中出现的次数与其语义重要性正相关。这种从布尔逻辑到整数统计的转变，使得模型能够区分"重要关键词"与"普通修饰词"。

举个例子，假设我们有以下三句话：

"我要去北京"
"想想就开心"
"我要去北京想想就开心"
基于之前的词表 ["我", "要", "去", "北京", "想想", "就", "开心"]，

我们可以构建一个词袋模型，表示如下：

| 文档编号 | 事儿 | 关注 | 就   | 开心 | 想想 | 我要 | 架构师 | 看   | 那些 |
| :------- | :--- | :--- | :--- | :--- | :--- | :--- | :----- | ---- | ---- |
| 1        | 1    | 1    | 0    | 0    | 0    | 1    | 1      | 0    | 1    |
| 2        | 0    | 0    | 1    | 1    | 1    | 0    | 0      | 0    | 0    |
| 3        | 1    | 0    | 1    | 1    | 1    | 1    | 1      | 1    | 1    |

可以看到，每一行表示一个文档，每一列表示一个词，值表示该词在文档中出现的次数。

代码示例：

```
		print("CountVectorizer")
    from sklearn.feature_extraction.text import CountVectorizer
    import jieba

    # 构建语料库
    corpus = [
        "我要去北京",
        "想想就开心",
        "我要去北京想想就开心"
    ]

    # 定义分词器函数
    def chinese_tokenizer(text):
        return jieba.cut(text)

    # 初始化词袋模型（指定分词器）
    vectorizer = CountVectorizer(tokenizer=chinese_tokenizer)

    # 训练并转换语料库
    X = vectorizer.fit_transform(corpus)

    # 输出词表和对应的词袋矩阵
    print("词表：", vectorizer.get_feature_names_out())
    print("词袋矩阵：\n", X.toarray())
```

输出结果：

```
CountVectorizer
/Users/lucas/Library/Python/3.9/lib/python/site-packages/sklearn/feature_extraction/text.py:517: UserWarning: The parameter 'token_pattern' will not be used since 'tokenizer' is not None'
  warnings.warn(
Building prefix dict from the default dictionary ...
Dumping model to file cache /var/folders/dy/yyk14n414z18v4vgl3npn2440000gn/T/jieba.cache
词表： ['北京' '去' '就' '开心' '想想' '我要']
词袋矩阵：
 [[1 1 0 0 0 1]
 [0 0 1 1 1 0]
 [1 1 1 1 1 1]]
Loading model cost 0.256 seconds.
Prefix dict has been built successfully.
```



核心优势解析

简单易用：词袋模型实现简单，易于理解和使用。

高效性：对于小规模数据集，词袋模型能够快速生成特征向量。

兼容性强：生成的稀疏矩阵可以直接用于机器学习算法，如逻辑回归、支持向量机等。

无序性：忽略词序后，模型对短文本的处理表现较好。



缺陷

忽略语义信息：词袋模型完全忽略了词语之间的顺序关系，可能会丢失上下文信息。例如，“我喜欢你”和“你不喜欢我”会被视为相同的向量。

高维稀疏性：当词表很大时，生成的特征向量会非常稀疏，导致计算效率低下。

无法捕捉相似性：不同词之间没有相似性度量，例如“开心”和“快乐”会被视为完全不同的词。



业务价值

有人说，词袋模型太古老了，现在业务中已经没有价值了。

非也非也。即便是这么古老的技术也是有意义的。

比如从直播中检测是否有人用录播冒充直播的情况：

输入是一段超长文本(重复录播可以做到24/7的不间断直播)，输出是是否存在录播重复True/False。

录播检测场景的技术选型逻辑：
1. 编辑距离：O(n²)时间复杂度，对10分钟直播流（约5000字）需计算2500万次对比
2. 词袋模型：滑动窗口内词频统计 → 向量化 → 余弦相似度计算，时间复杂度降为O(n)



技术对比

| 方案                | 时间复杂度 | 准确率 | 适用场景       |
| :------------------ | :--------- | :----- | :------------- |
| 编辑距离            | O(n²)      | 98%    | 短文本精准匹配 |
| 词袋模型+余弦相似度 | O(n)       | 92%    | 实时流媒体检测 |



### TF-IDF

TF-IDF 是一种统计方法，用于评估一个词在文档或语料库中的重要性。

其核心思想是：如果某个词在一个文档中频繁出现，但在整个语料库中很少出现，则该词可能对这个文档来说是非常重要的。



TF（Term Frequency, 词频）：衡量一个词在文档中出现的频率。可以通过简单计数、归一化等方式计算。

TF(词) = (该词在文档中出现的次数) / (文档中所有词的数量)

举个例子，如果一个词在一篇有100个词的文档里出现了5次，那么这个词的词频就是 5/100=0.055/100=0.05。



IDF（Inverse Document Frequency, 逆文档频率）：衡量一个词的普遍重要性。如果一个词在很多文档中都出现过，那么它可能不是一个好的区分者。

IDF(词) = log(总文档数 / 包含该词的文档数量)

这里的 log 是以 e 为底的对数函数。例如，如果有1000篇文档，其中10篇包含某个词，则 IDF = log(1000/10)=log(100)log(1000/10)=log(100)。



TF-IDF：结合了上述两种指标，旨在通过降低在所有文档中都很常见的词汇的重要性来突出那些有助于区分文档的词汇。

TF-IDF = TF × IDF

这意味着，一个词在一个特定文档中的重要性不仅取决于它在这个文档中出现的频率，还取决于它在整个文档集合中的普遍程度。



代码示例：

```
    print("TfidfVectorizer")
    from sklearn.feature_extraction.text import TfidfVectorizer

    # 构建语料库
    corpus = [
        "我要去北京",
        "想想就开心",
        "我要去北京想想就开心"
    ]
    # 初始化TF-IDF模型
    vectorizer = TfidfVectorizer()
    # 训练并转换语料库
    X = vectorizer.fit_transform(corpus)
    # 输出词表和对应的TF-IDF矩阵
    print("词表：", vectorizer.get_feature_names_out())
    print("TF-IDF矩阵：\n", X.toarray())
```

输出结果：

```
TfidfVectorizer
词表： ['北京' '去' '就' '开心' '想想' '我要']
TF-IDF矩阵：
 [[0.57735027 0.57735027 0.         0.         0.         0.57735027]
 [0.         0.         0.57735027 0.57735027 0.57735027 0.        ]
 [0.40824829 0.40824829 0.40824829 0.40824829 0.40824829 0.40824829]]
Loading model cost 0.223 seconds.
Prefix dict has been built successfully.
```



核心优势解析

强调关键信息：相比于简单的词袋模型，TF-IDF 更加关注那些能够区分不同文档的关键词汇。

减少常见词影响：通过 IDF 部分，减少了像“的”、“是”这样的高频但无实际意义的词汇的影响。

适应性强：可以应用于各种类型的文本数据，并且不需要复杂的参数调整。



缺陷

忽略上下文：尽管TF-IDF能有效识别关键词，但它仍然忽略了词语之间的顺序和上下文关系。

维度灾难：对于非常大的语料库，生成的特征向量维度非常高，可能导致计算效率问题。

无法捕捉同义词：不同的词即使具有相似的含义，也会被视为完全不同的实体。



业务价值

TF-IDF 在如今仍然很常用，如低成本策略大多采用TF-IDF做特征工程，然后接一个分类器做文本分类，这种业务实际中太多了。

较常用的，TF-IDF (特征) + XGBoost/SVM（分类器）= 分类模型

虽然效果上限低于GPU类模型(RNN和预训练模型)，但是成本超低，响应超快，仍然有不少企业采用这种解决方案。



### 技术对比矩阵

| 评估维度     | One-Hot        | 词袋模型   | TF-IDF       |
| :----------- | :------------- | :--------- | :----------- |
| 语义区分能力 | ❌              | 词频差异   | 跨文档重要性 |
| 空间复杂度   | O(V)           | O(V)       | O(V)         |
| 实时计算性能 | O(1)           | O(n)       | O(n log n)   |
| 上下文感知   | ❌              | ❌          | ❌            |
| 最佳适用场景 | 小规模枚举特征 | 短文本聚类 | 信息检索     |



