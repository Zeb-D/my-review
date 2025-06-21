本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

在深度学习领域或者大模型领域中，光说不练真的很容易遗忘，就跟我一样入门rust三四年，结果还在入门。

接着上篇文章，这次我们看下如何把词向量化后，放入到学习中。



### 概念

循环神经网络（Recurrent Neural Network，RNN）是一种基于顺序或时间序列数据进行训练的深度神经网络，可根据序列输入做出序列预测或结论1。以下是关于它的详细介绍1：

#### 特点

- **记忆特性**：与传统神经网络不同，RNN 具有 “记忆” 功能，能从先前输入中获取信息来影响当前的输入和输出，其输出取决于序列中的先验元素。通过在每个时间步长维护隐藏状态来跟踪上下文，隐藏状态充当存储有关先前输入信息的记忆。
- **参数共享**：循环网络在网络的每一层共享相同的权重参数，而前馈网络在每个节点上具有不同的权重。这些权重在反向传播和梯度下降过程中进行调整，以促进强化学习。



### 工作原理

- **输入处理**：在每个时间步长，RNN 接收当前输入（如句子中的单词、时间序列中的数据点）以及来自前一个时间步长的隐藏状态，将两者结合进行处理。
- **激活函数**：应用激活函数于网络中每一层神经元的输出，引入非线性，使网络能够处理非线性问题，控制神经元输出幅度，防止值在传递过程中过大或过小。常见的激活函数有 Sigmoid 函数、Tanh（双曲正切）函数、ReLU（修正线性单元）及其变体等。
- **输出生成**：基于当前输入和隐藏状态，通过计算生成当前时间步长的输出，同时更新隐藏状态，传递到下一个时间步长，形成反馈循环。



#### 类型

- **标准版循环神经网络（RNN）**：每一时间步长的输出都取决于当前输入和上一时间步长的隐藏状态，擅长处理具有短期依赖性的简单任务，但存在梯度消失等问题，难以学习长期依赖关系。
- **双向循环神经网络（BRNN）**：单向 RNN 只能利用先前输入数据，而 BRNN 还可以拉取未来的数据，从而提高预测准确性。
- **长短期记忆（LSTM）**：为解决梯度消失和长期依赖问题而提出，在人工神经网络的隐藏层中设有 “单元”，包含输入门、输出门和遗忘门，通过控制信息流来更好地处理长期依赖关系。
- **门控循环单元（GRU）**：类似于 LSTM，可解决 RNN 模型的短期记忆问题。它不使用 “元胞状态” 调节信息，而是使用隐藏状态，通过重置门和更新门控制要保留的信息，架构更简洁，计算效率高，训练速度快，适用于实时或资源受限的应用程序。
- **编码器 - 解码器 RNN**：通常用于序列到序列任务，如机器翻译。编码器将输入序列处理为固定长度的向量（上下文），解码器则使用该上下文生成输出序列，但固定长度的上下文向量可能成为瓶颈，尤其是对于较长的输入序列。



#### 应用

- **自然语言处理（NLP）**：如语言翻译、情感分析、语音识别、图像字幕、文本生成等任务。
- **时间序列预测**：根据过去的时间序列数据，如每日洪水、潮汐和气象数据等，预测未来的值。
- **其他领域**：还可应用于处理传感器数据以检测短时间内的异常、音乐生成、情绪分类等场景。



尽管 RNN 在人工智能中的使用有所减少，尤其是随着转换器模型的兴起，但它在一些特定环境中仍有应用，特别是在其序列性质和记忆机制能发挥作用的场景中2。



### 实践

这次我们就用jaychou的歌词进行训练，然后把参数保存到模型文件，最后来生成一首大家自己的歌曲。



#### 歌词预处理

我们用上篇文章的中文分词器jieba来切割：

```python

# 数据预处理函数 - 更新为处理 ZIP 文件
def preprocess_data(file_path):
    text = ""

    # 检查是否为 ZIP 文件
    if file_path.endswith('.zip'):
        with zipfile.ZipFile(file_path, 'r') as zip_ref:
            # 获取所有文件名
            file_names = zip_ref.namelist()

            # 读取所有文本文件内容
            for name in file_names:
                if name.endswith('.txt'):
                    with zip_ref.open(name) as f:
                        # 尝试不同的编码方式读取文件
                        try:
                            content = f.read().decode('utf-8')
                        except UnicodeDecodeError:
                            content = f.read().decode('gbk')
                        text += content
    else:
        # 处理普通文本文件
        with open(file_path, 'r', encoding='utf-8') as f:
            text = f.read()

    # 使用 jieba 分词
    words = jieba.lcut(text)

    # 创建词到索引和索引到词的映射
    unique_words = sorted(list(set(words)))
    word_to_idx = {word: i for i, word in enumerate(unique_words)}
    idx_to_word = {i: word for i, word in enumerate(unique_words)}
    vocab_size = len(unique_words)

    print(f"语料库词数: {len(words)}")
    print(f"词汇表大小: {vocab_size}")

    return words, word_to_idx, idx_to_word, vocab_size

```

我们用一个对象LyricsDataset包装歌词的张量tensor：

```python

class LyricsDataset(Dataset):
    def __init__(self, words, seq_length, word_to_idx, idx_to_word, vocab_size):
        self.words = words
        self.seq_length = seq_length
        self.word_to_idx = word_to_idx
        self.idx_to_word = idx_to_word
        self.vocab_size = vocab_size

        # 计算可用的序列数量
        self.num_sequences = len(words) - seq_length - 1

    def __len__(self):
        return self.num_sequences

    def __getitem__(self, idx):
        # 获取输入序列和目标序列
        input_seq = self.words[idx:idx + self.seq_length]
        target_seq = self.words[idx + 1:idx + self.seq_length + 1]

        # 将词转换为索引
        input_idx = [self.word_to_idx[word] for word in input_seq]
        target_idx = [self.word_to_idx[word] for word in target_seq]

        # 转换为 PyTorch 张量
        input_tensor = torch.tensor(input_idx, dtype=torch.long)
        target_tensor = torch.tensor(target_idx, dtype=torch.long)

        return input_tensor, target_tensor
```

如果大家看了上篇文章，其实发现处理方式差不多，同时我们这里也用到了设计模式的组合模式。



#### 定义神经网络模型

这里我们采用RNN类型，大家要可以改成其他RNN类型，效果大同小异。

```python

class RNNModel(nn.Module):
    def __init__(self, vocab_size, embed_size, hidden_size, num_layers, dropout=0.5):
        super(RNNModel, self).__init__()
        self.vocab_size = vocab_size
        self.embed_size = embed_size
        self.hidden_size = hidden_size
        self.num_layers = num_layers

        # 定义网络层
        self.embedding = nn.Embedding(vocab_size, embed_size)
        self.rnn = nn.RNN(embed_size, hidden_size, num_layers,
                          batch_first=True, dropout=dropout)
        self.fc = nn.Linear(hidden_size, vocab_size)

    def forward(self, x, hidden):
        # 嵌入层
        embedded = self.embedding(x)

        # RNN 层
        output, hidden = self.rnn(embedded, hidden)

        # 全连接层
        output = self.fc(output)

        return output, hidden

    def init_hidden(self, batch_size):
        # 初始化隐藏状态
        return torch.zeros(self.num_layers, batch_size, self.hidden_size, device=device)

```



#### 训练逻辑

data_loader：词向量的input、output，LyricsDataset实现了该接口

model：我们上面定义的神经网络模型

criterion：损失函数，用于衡量模型预测的概率分布与真实标签之间的差异。模型训练的目标就是最小化这个损失值，使得模型的预测结果尽可能接近真实标签。

optimizer：优化器，主要和梯度有关系，后面专门讲下这几块的概念与实践。

训练的目的：通过多次训练，将损失函数逼近，最终得到神经网络的参数(大模型文件)。

这个方法很重要，要保证训练过程有更高的学习率以及质量。

```python

# 使用tqdm显示进度条，优化训练循环
def train(model, data_loader, criterion, optimizer, scheduler, epochs, device, save_path='models/', start_epoch=0,
          patience=3, ):
    # 创建保存模型的目录
    if not os.path.exists(save_path):
        os.makedirs(save_path)

    model.train()
    best_loss = float('inf')
    early_stop_counter = 0

    for epoch in range(start_epoch, epochs):
        cur_loss = 0
        start_time = time.time()
        hidden = None

        # 使用tqdm显示进度条
        progress_bar = tqdm(enumerate(data_loader), total=len(data_loader))
        for i, (inputs, targets) in progress_bar:
            inputs, targets = inputs.to(device), targets.to(device)
            batch_size = inputs.size(0)

            # 为每个批次创建新的隐藏状态
            if hidden is None or hidden[0].size(1) != batch_size:
                hidden = model.init_hidden(batch_size)
            else:
                # 分离隐藏状态，防止梯度累积
                hidden = (hidden[0].detach(), hidden[1].detach())

            # 前向传播
            outputs, hidden = model(inputs, hidden)
            loss = criterion(outputs.view(-1, model.vocab_size), targets.view(-1))

            # 反向传播和优化
            optimizer.zero_grad()
            loss.backward()
            # 梯度裁剪，防止梯度爆炸
            nn.utils.clip_grad_norm_(model.parameters(), 0.5)
            optimizer.step()

            cur_loss = loss.item()

            # 更新进度条
            progress_bar.set_description(
                f'Epoch-b {epoch + 1}/{epochs}, Loss: {loss.item():.4f}, cur_loss: {cur_loss:.4f}')

        # 计算平均损失
        avg_loss = cur_loss / len(data_loader)

        # 学习率调度
        scheduler.step(avg_loss)

        # 打印训练信息
        elapsed = time.time() - start_time
        print(f'Epoch-1: {epoch + 1}/{epochs} | Loss: {avg_loss:.4f} | '
              f'Perplexity: {math.exp(avg_loss):.2f} | Time: {elapsed:.2f}s')

        # 保存最佳模型和最新模型
        if avg_loss < best_loss:
            best_loss = avg_loss
            torch.save(model.state_dict(), f'{save_path}model-rnn.pt')
            print(f'Best model saved with loss: {avg_loss:.4f}，best_loss: {best_loss:.4f}')

        # 只保存最新的检查点，覆盖之前的
        torch.save({
            'epoch': epoch + 1,
            'model_state_dict': model.state_dict(),
            'optimizer_state_dict': optimizer.state_dict(),
            'scheduler_state_dict': scheduler.state_dict(),
            'loss': avg_loss,
        }, f'{save_path}checkpoint-rnn.pt')

        # 早停检查
        if avg_loss >= best_loss:
            early_stop_counter += 1
            print(f'Early stopping counter: {early_stop_counter}/{patience}')
            if early_stop_counter >= patience:
                print(f'Early stopping triggered after {epoch + 1} epochs')
                break
        else:
            early_stop_counter = 0

    # 加载最佳模型
    model.load_state_dict(torch.load(f'{save_path}model-rnn.pt'))
    return model
```



#### 推理生成歌词

word_to_idx, idx_to_word 这两个都是在词向量化预处理得到的词与下标。

最关键的地方是seed_words会作为开始的记忆开始前向传播。

在文本生成模型中，**temperature（温度）** 是一个重要的超参数，用于控制生成文本的随机性和创造性程度。它主要通过调整 softmax 函数的输出分布来实现这一控制。

设置低温度，像一个严格的模仿者，复制训练数据中的常见模式。

高温度，像一个充满创意的艺术家，可能会产生新颖但有时不合理的内容。 

```

def generate_text(model, word_to_idx, idx_to_word, seed_text, max_length=200, temperature=1.0):
    model.eval()

    # 将种子文本分词并转换为索引
    seed_words = jieba.lcut(seed_text)
    input_idx = []

    # 处理每个词，使用未知词标记或随机词替换不存在的词
    for word in seed_words:
        if word in word_to_idx:
            input_idx.append(word_to_idx[word])
        else:
            print(f"警告: 词 '{word}' 不在词汇表中")
            # 尝试使用 <UNK> 标记，如果词汇表中有的话
            if '<UNK>' in word_to_idx:
                input_idx.append(word_to_idx['<UNK>'])
            else:
                # 没有 <UNK> 标记时，使用随机词
                input_idx.append(random.randint(0, len(word_to_idx) - 1))

    # 如果处理后仍然没有有效词，使用随机词开始
    if not input_idx:
        print("警告: 种子文本处理后没有有效词，使用随机词开始")
        input_idx = [random.randint(0, len(word_to_idx) - 1)]

    # 初始化输入张量
    input_tensor = torch.tensor(input_idx, dtype=torch.long).unsqueeze(0).to(device)

    # 初始化隐藏状态
    hidden = model.init_hidden(1)

    # 生成文本
    generated_text = seed_text

    with torch.no_grad():
        for i in range(max_length):
            # 前向传播
            output, hidden = model(input_tensor, hidden)

            # 应用温度参数调整预测分布
            output = output[:, -1, :] / temperature
            probs = torch.softmax(output, dim=1)

            # 采样下一个词索引
            next_idx = torch.multinomial(probs, num_samples=1).item()
            next_word = idx_to_word[next_idx]

            # 添加到生成的文本中
            generated_text += next_word

            # 关键改进：将新生成的词添加到输入序列中
            input_idx.append(next_idx)
            input_tensor = torch.tensor(input_idx, dtype=torch.long).unsqueeze(0).to(device)

    return generated_text

```



#### 主函数

把上面的逻辑串起来。

在训练train方法，每一个epoch大约要13s的耗时，当前设置了30个批次训练，言外之意，大家可以设置小些。

另外，这个支持第二次运行使用上次的模型结果，也支持接着训练，看retrain函数。

```python
# 主函数
def main(retrain=False):
    # 数据预处理
    file_path = 'jaychou_lyrics.txt.zip'  # 请替换为实际的歌词 ZIP 文件路径
    words, word_to_idx, idx_to_word, vocab_size = preprocess_data(file_path)

    # 创建数据集和数据加载器
    seq_length = 100
    batch_size = 64
    dataset = LyricsDataset(words, seq_length, word_to_idx, idx_to_word, vocab_size)
    data_loader = DataLoader(dataset, batch_size=batch_size, shuffle=True)

    # 模型参数
    embed_size = 256
    hidden_size = 512
    num_layers = 2
    dropout = 0.5

    # 初始化模型
    model = RNNModel(vocab_size, embed_size, hidden_size, num_layers, dropout).to(device)

    # 定义损失函数和优化器
    criterion = nn.CrossEntropyLoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
    # 添加学习率调度
    scheduler = ReduceLROnPlateau(optimizer, mode='min', factor=0.5, patience=2, verbose=True)

    # 检查是否有保存的模型
    save_path = 'models/'
    start_epoch = 0
    epochs = 30
    if os.path.exists(f"{save_path}checkpoint-rnn.pt"):
        print("loading model...\n")
        checkpoint = torch.load(f"{save_path}checkpoint-rnn.pt")
        model.load_state_dict(checkpoint['model_state_dict'])
        optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
        scheduler.load_state_dict(checkpoint['scheduler_state_dict'])
        start_epoch = checkpoint['epoch']
        print(f"从 epoch {start_epoch} 加载训练")
        print(f"之前的最佳损失: {checkpoint['loss']:.4f}")
        print(f"Loaded existing model from {save_path}")
    else:
        retrain = True  # 需要重新训练
    if retrain:
        print("training model...\n")
        model = train(model, data_loader, criterion, optimizer, scheduler, epochs, device, save_path, start_epoch)

    # 生成歌词
    seed_text = "窗外的麻雀"  # 可以替换为你喜欢的任意歌词开头
    generated_lyrics = generate_text(model, word_to_idx, idx_to_word, seed_text,
                                     max_length=300, temperature=0.8)

    print("\n生成的歌词:")
    print(generated_lyrics)


```



#### 训练的日志

```
/Users/lucas/.pyenv/versions/3.9.18/bin/python /Users/lucas/PycharmProjects/ai/deepLearning/jaychou_lyrics_generator_RNN.py 
Using MPS (Metal Performance Shaders) for GPU acceleration
Building prefix dict from the default dictionary ...
Loading model from cache /var/folders/dy/yyk14n414z18v4vgl3npn2440000gn/T/jieba.cache
Loading model cost 0.211 seconds.
Prefix dict has been built successfully.
语料库词数: 43317
词汇表大小: 5703
training model...

Epoch-b 1/30, Loss: 0.2065, cur_loss: 0.2065: 100%|██████████| 676/676 [00:53<00:00, 12.75it/s]
Epoch-1: 1/30 | Loss: 0.0003 | Perplexity: 1.00 | Time: 53.02s
Best model saved with loss: 0.0003，best_loss: 0.0003
  0%|          | 0/676 [00:00<?, ?it/s]Early stopping counter: 1/3
Epoch-b 2/30, Loss: 0.1210, cur_loss: 0.1210: 100%|██████████| 676/676 [00:52<00:00, 12.76it/s]
Epoch-1: 2/30 | Loss: 0.0002 | Perplexity: 1.00 | Time: 52.96s
Best model saved with loss: 0.0002，best_loss: 0.0002
Early stopping counter: 2/3
Epoch-b 3/30, Loss: 0.1161, cur_loss: 0.1161: 100%|██████████| 676/676 [00:53<00:00, 12.71it/s]
Epoch-1: 3/30 | Loss: 0.0002 | Perplexity: 1.00 | Time: 53.20s
Best model saved with loss: 0.0002，best_loss: 0.0002
Early stopping counter: 3/3
Early stopping triggered after 3 epochs

生成的歌词:
窗外的麻雀 让我把记忆结成冰
别融化了眼泪　
你妆都花了要我怎么记得
记得你叫我忘了吧　
记得你叫我忘了吧
你说你会哭　
不是因为在乎
不是因为在乎朦胧的时间　
我们溜了多远
冰刀划的　
圈起了谁改变
如果再重来　
会不会稍嫌狼狈
爱是不是不开口才珍贵
再给我两分钟　
让我把记忆结成冰
别融化了眼泪　
你妆都花了要我怎么记得
记得你叫我忘了吧　
记得你叫我忘了吧
你说你会哭　
不是因为在乎
不是因为在乎
我一拳打飞一幕幕的回忆散在月光
一截老老的老姜
一段旧旧的旧时光
我可以给你们一张签名照拿去想象
我说啊 屏风就该遮冰霜
屋檐就该挡月光
江湖就该开扇窗
平剧就该耍花枪
扎下马步我不摇晃
闷了慌了倦了我就穿上功夫装
我不卖豆腐 豆腐  豆腐 豆腐 
我在武功学校里学的那叫功夫
功夫 功夫  功夫 功夫 
赶紧穿上旗袍
免得你说我吃你豆腐
你就像豆腐 豆腐  豆腐 豆腐 
吹弹可破的肌肤在试练我功夫
功夫

Process finished with exit code 0

```





### 可以提升的点

留给大家一些作业，大家可以通过chatGPT这种进行完成：

1、多进程批量训练，提示pytorch支持多个进程训练。

2、评价RNN优缺点，你觉得RNN的试用场景？

3、如果你想怎么改造本案例？
