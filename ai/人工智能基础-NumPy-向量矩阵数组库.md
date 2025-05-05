本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

python三大库numpy，pandas以及matplotlib在人工智能领域有广泛的营运。下面我将介绍一些关于Numpy的一些简单教程

Numpy是什么?

NumPy (Numeric Python)提供了许多高级的数值编程工具，如:矩阵数据类型、矢量处理，以及精密的运算库。专为进行严格的数字处理而产生。

numpy的优点在于，他的大部分代码都是用c编写的，所以它在实现起来比python的代码有了更高的性能。



Numpy主要用于数据分析，整体的模式有点类似于小型的R，在读取和处理数据上略占优势，但是R的强大在于其统计分析和可视化。



### 基本用法

#### 数组

通过传递给`np.array()`函数一个list对象，来创建一个数组

```
		# 构建数组
    np.array([1,2,3])
    print(np.ones(3,dtype=str)) # ['1' '1' '1']
    print(np.random.random(3))  # [0.19142482 0.55526267 0.62837514]
```



#### 矩阵

与数组的构建写法相同，但是不同的是矩阵可以包含二维关系，同样包含ones()、zeros() 和 random.random() 快速方法

```
    # 构建矩阵
    print(np.array([[1, 2], [3, 4]]))
    print(np.zeros((3, 2),dtype=int))
```



#### 提取子集

使用下标和切片提取，python的下标是从0开始的：

```
		print("提取子集")
    data = np.array([[1, 2], [3, 4], [5, 6]])
    print(data)
    print(data[0, 1]) # 0行1列 # 输出 2
    # print(data[3, 2]) # 3行2列 # 直接报错
    print(data[1:4]) # 输出1到4行元素，越界不会报错
    print(data[1:3, ])
    print(data[0:2, 0]) #输出0到2行的 0列元素
```



#### 数组转置与reshape()

转置：行变成列，列变成行

```
		print("数组转置与reshape()")
    data = np.array([[1, 2], [3, 4], [5, 6]])
    print(data)
    print(data.T) # 你可以理解行与列的下标互换
    
    print("reshape")
    data = np.array([1, 2, 3, 4, 5, 6])
    print(data.reshape(2, 3)) # 将一维数组变成2行3列数组
    # print(data.reshape(2, 2)) # 元素剩余会报错
    # print(data.reshape(3, 3)) # 元素不够会报错
```



### 数组运算

#### 加减乘除等数学运算

```
		print("矩阵可以进行加减乘除等数学运算:")
    data = np.array([[1, 2], [3, 4]])
    ones = np.ones((2,2))  # 1行2列

    ad = data + ones  # 对应下标进行相加
    print(ad)
    print(data - ones)  # 相减
    print(data * data)  # 逐元素相乘
    print(data / data)
    print(data * 1.6)
```



#### 最小值/最大值/平均值

```
		print("最小值/最大值/平均值等")
    print(data.max())
    print(data.max(axis=1))  # 按行分组计算最大值
    print(data.min())
    print(data.sum())
    print("所有元素的乘积:", data.prod())  # 1*2*3*4 = 24
    print("标准差（std）:", data.std())  # √(方差) ≈ 1.118
    print("方差（var）:", data.var())  # 方差 = 1.25
    print("矩阵平方:", np.square(data))  # 矩阵平方
```

上面这些方法都有参数 `axis=0`表示按列运算，`axis=1`表示按行运算

> 更多的运算见：https://jakevdp.github.io/PythonDataScienceHandbook/02.04-computation-on-arrays-aggregates.html





#### 点积运算(dot)

这个运算看起来比较像线性代数里面的两个矩阵相乘：

```
		print("点积运算(dot)")
    data1 = np.array([1, 2, 3])
    data2 = np.array([[1, 10], [100, 1000], [10000, 100000]])
    print(data1.dot(data2))  # 输出线性代数的矩阵相乘，不是简单上面的逐元素相乘*

```

公式要求行数必须等于另外一个矩阵的列数，即矩阵m * n 和矩阵n * j ，最终得到另外一个矩阵的m * j 维数组。



### 高级运算

除了向量和矩阵支持运算外，公式的使用也是 NumPy 成为宠儿的原因之一。

例如在解决回归问题中监督式机器学习的核心公式：均方差公式

```
		print("高级运算")
    print("均方差公式")
    n = 6
    predictions = np.array([[1, 2, 3], [4, 5, 6]])
    labels = np.ones((2, 3))
    error = (1 / n) * np.sum(np.square(predictions - labels)) # np.square 为矩阵的平方
    print(error)
```



### 函数汇总

上面只是部份示例，下面从官网找到了一个比较全的示例：

https://numpy.org/doc/stable/reference/ufuncs.html

#### 1. 数学运算函数

| 函数            | 说明                           | 示例                    | 输出示例         |
| :-------------- | :----------------------------- | :---------------------- | :--------------- |
| `np.exp(x)`     | 自然指数 ex*e**x*              | `np.exp([1, 2])`        | `[2.718, 7.389]` |
| `np.expm1(x)`   | ex−1*e**x*−1（高精度小值计算） | `np.expm1([0, 1e-10])`  | `[0., 1e-10]`    |
| `np.log(x)`     | 自然对数 ln⁡(x)ln(*x*)          | `np.log([1, np.e])`     | `[0., 1.]`       |
| `np.log10(x)`   | 以 10 为底的对数               | `np.log10([1, 100])`    | `[0., 2.]`       |
| `np.log2(x)`    | 以 2 为底的对数                | `np.log2([1, 8])`       | `[0., 3.]`       |
| `np.log1p(x)`   | ln⁡(1+x)ln(1+*x*)（高精度小值） | `np.log1p([0, 1e-10])`  | `[0., 1e-10]`    |
| `np.sqrt(x)`    | 平方根 x*x*                    | `np.sqrt([4, 9])`       | `[2., 3.]`       |
| `np.square(x)`  | 平方 x2*x*2                    | `np.square([2, 3])`     | `[4, 9]`         |
| `np.abs(x)`     | 绝对值                         | `np.abs([-1, 2])`       | `[1, 2]`         |
| `np.ceil(x)`    | 向上取整                       | `np.ceil([1.2, -1.7])`  | `[2., -1.]`      |
| `np.floor(x)`   | 向下取整                       | `np.floor([1.7, -1.2])` | `[1., -2.]`      |
| `np.round(x)`   | 四舍五入                       | `np.round([1.4, 1.6])`  | `[1., 2.]`       |
| `np.sign(x)`    | 符号函数（-1, 0, 1）           | `np.sign([-5, 0, 5])`   | `[-1, 0, 1]`     |
| `np.sin(x)`     | 正弦（弧度制）                 | `np.sin(np.pi/2)`       | `1.0`            |
| `np.cos(x)`     | 余弦（弧度制）                 | `np.cos(0)`             | `1.0`            |
| `np.tan(x)`     | 正切（弧度制）                 | `np.tan(np.pi/4)`       | `≈1.0`           |
| `np.arcsin(x)`  | 反正弦                         | `np.arcsin(0.5)`        | `≈0.5236`        |
| `np.arccos(x)`  | 反余弦                         | `np.arccos(0.5)`        | `≈1.0472`        |
| `np.arctan(x)`  | 反正切                         | `np.arctan(1)`          | `≈0.7854`        |
| `np.sinh(x)`    | 双曲正弦                       | `np.sinh(0)`            | `0.0`            |
| `np.cosh(x)`    | 双曲余弦                       | `np.cosh(0)`            | `1.0`            |
| `np.tanh(x)`    | 双曲正切                       | `np.tanh(0)`            | `0.0`            |
| `np.deg2rad(x)` | 角度转弧度                     | `np.deg2rad(180)`       | `≈3.1416`        |
| `np.rad2deg(x)` | 弧度转角度                     | `np.rad2deg(np.pi)`     | `180.0`          |

------

#### 2. 比较运算函数

| 函数                     | 说明           | 示例                               | 输出示例        |
| :----------------------- | :------------- | :--------------------------------- | :-------------- |
| `np.equal(x, y)`         | 逐元素相等比较 | `np.equal([1, 2], [1, 3])`         | `[True, False]` |
| `np.not_equal(x, y)`     | 逐元素不等比较 | `np.not_equal([1, 2], [1, 3])`     | `[False, True]` |
| `np.greater(x, y)`       | 逐元素 `x > y` | `np.greater([2, 1], [1, 2])`       | `[True, False]` |
| `np.greater_equal(x, y)` | 逐元素 `x ≥ y` | `np.greater_equal([2, 1], [1, 1])` | `[True, True]`  |
| `np.less(x, y)`          | 逐元素 `x < y` | `np.less([1, 2], [2, 1])`          | `[True, False]` |
| `np.less_equal(x, y)`    | 逐元素 `x ≤ y` | `np.less_equal([1, 2], [1, 1])`    | `[True, False]` |

------

#### 3. 逻辑运算函数

| 函数                   | 说明           | 示例                                           | 输出示例        |
| :--------------------- | :------------- | :--------------------------------------------- | :-------------- |
| `np.logical_and(x, y)` | 逐元素逻辑与   | `np.logical_and([True, False], [True, True])`  | `[True, False]` |
| `np.logical_or(x, y)`  | 逐元素逻辑或   | `np.logical_or([True, False], [False, False])` | `[True, False]` |
| `np.logical_not(x)`    | 逐元素逻辑非   | `np.logical_not([True, False])`                | `[False, True]` |
| `np.logical_xor(x, y)` | 逐元素逻辑异或 | `np.logical_xor([True, False], [True, True])`  | `[False, True]` |

------

#### 4. 统计运算函数

| 函数                  | 说明         | 示例                           | 输出示例    |
| :-------------------- | :----------- | :----------------------------- | :---------- |
| `np.sum(x)`           | 所有元素求和 | `np.sum([1, 2, 3])`            | `6`         |
| `np.mean(x)`          | 所有元素均值 | `np.mean([1, 2, 3])`           | `2.0`       |
| `np.median(x)`        | 中位数       | `np.median([1, 3, 2])`         | `2.0`       |
| `np.std(x)`           | 标准差       | `np.std([1, 2, 3])`            | `≈0.816`    |
| `np.var(x)`           | 方差         | `np.var([1, 2, 3])`            | `≈0.666`    |
| `np.min(x)`           | 最小值       | `np.min([1, 2, 3])`            | `1`         |
| `np.max(x)`           | 最大值       | `np.max([1, 2, 3])`            | `3`         |
| `np.percentile(x, q)` | q 分位数     | `np.percentile([1, 2, 3], 50)` | `2.0`       |
| `np.cumsum(x)`        | 累积和       | `np.cumsum([1, 2, 3])`         | `[1, 3, 6]` |
| `np.cumprod(x)`       | 累积积       | `np.cumprod([1, 2, 3])`        | `[1, 2, 6]` |

------

#### 5. 线性代数函数

| 函数               | 说明             | 示例                              | 输出示例                 |
| :----------------- | :--------------- | :-------------------------------- | :----------------------- |
| `np.dot(A, B)`     | 矩阵乘法         | `np.dot([1, 2], [[3], [4]])`      | `11`                     |
| `np.matmul(A, B)`  | 矩阵乘法（推荐） | `np.matmul([[1, 2]], [[3], [4]])` | `[[11]]`                 |
| `np.linalg.inv(A)` | 矩阵求逆         | `np.linalg.inv([[1, 2], [3, 4]])` | `[[-2, 1], [1.5, -0.5]]` |
| `np.linalg.det(A)` | 行列式           | `np.linalg.det([[1, 2], [3, 4]])` | `-2.0`                   |
| `np.linalg.eig(A)` | 特征值和特征向量 | `np.linalg.eig([[1, 2], [2, 1]])` | 返回特征值和向量         |
| `np.linalg.svd(A)` | 奇异值分解       | `np.linalg.svd([[1, 2], [3, 4]])` | 返回 U, Σ, V             |

------

#### 6. 数组操作函数

| 函数                    | 说明       | 示例                                      | 输出示例                         |
| :---------------------- | :--------- | :---------------------------------------- | :------------------------------- |
| `np.unique(x)`          | 返回唯一值 | `np.unique([1, 2, 2, 3])`                 | `[1, 2, 3]`                      |
| `np.sort(x)`            | 排序       | `np.sort([3, 1, 2])`                      | `[1, 2, 3]`                      |
| `np.argsort(x)`         | 排序索引   | `np.argsort([3, 1, 2])`                   | `[1, 2, 0]`                      |
| `np.clip(x, a, b)`      | 限制范围   | `np.clip([0, 5, 10], 1, 9)`               | `[1, 5, 9]`                      |
| `np.where(cond, x, y)`  | 条件选择   | `np.where([True, False], [1, 2], [3, 4])` | `[1, 4]`                         |
| `np.concatenate((a,b))` | 数组拼接   | `np.concatenate(([1, 2], [3, 4]))`        | `[1, 2, 3, 4]`                   |
| `np.split(x, indices)`  | 数组分割   | `np.split([1, 2, 3, 4], [2])`             | `[array([1, 2]), array([3, 4])]` |



总结

1. **数学运算**：覆盖指数、对数、三角函数等基本运算。
2. **比较与逻辑**：支持逐元素的布尔运算。
3. **统计计算**：提供均值、方差、分位数等统计量。
4. **线性代数**：矩阵乘法、求逆、特征值分解等。
5. **数组操作**：排序、去重、条件筛选等。



### **更多应用** 

#### 音频文件

一段音频可以存为一个一维数组，通过切片法可以剪辑任意一段：

> CD 质量的音频每秒包含 44,100 个样本，每个样本是-65535 到 65536 之间的整数。这意味着如果你有一个 10 秒的 CD 质量 WAVE 文件，你可以将它加载到长度为 10 * 44,100 = 441,000 的 NumPy 数组中



#### 图像Images

图像是像素大小 (高度 x 宽度) 的矩阵。

如果图像是**黑白的 (也称为灰度)** ，每个像素可以用一个数字表示 (通常介于 0 (黑色) 和 255 (白色) 之间)。想要裁剪图像左上角 10 x 10 像素的部分吗？使用numpy



如果图像是**彩色的**，那么每个像素用三个数字表示 —— 红色、绿色和蓝色各一个值。在这种情况下，我们需要一个三维 (因为每个单元格只能包含一个数字)。所以一个彩色图像是由一个3维数组表示的： (高 x 宽 x 3)



使用图的形式直观展示出来是不是容易理解多了~