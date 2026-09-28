# Transformer 英译中：从训练到「零 .py」标准 HuggingFace 发布与接入

> 之前 [​如何把自己训练的模型 上传到HF与接入文档](./HF_导出与接入文档.md) 这篇写的有点粗糙，没有按标准的HF格式来，本文以本仓库真实代码为准，讲解四件事：
> 
> 1. Transformer 在本文中的完整流转
> 2. 训练数据与核心训练代码步骤
> 3. 上传 HuggingFace 的标准 / 格式 / 要求与上传代码
> 4. 第三人如何下载并接入使用
> 
> 涉及文件：
> [`train_marian.py`](train_marian.py)、[`demo_inference.py`](demo_inference.py)、
> [`tools/tokenizer_utils.py`](tools/tokenizer_utils.py)、
> [`tokenizer/tokenization_transformer.py`](tokenizer/tokenization_transformer.py)、
> [`model/configuration_transformer.py`](model/configuration_transformer.py)、
> [`model/modeling_transformer.py`](model/modeling_transformer.py)。

---

## 0. 结论先行

本项目的**主线产物**是：用 **transformers 内置架构 `MarianMTModel`** 训练的英译中模型，
发布到 https://huggingface.co/chou-lucas/transformer-en-zh 形态为 零 `.py` 的标准 HF 仓库：

```
config.json              generation_config.json    model.safetensors
tokenizer_config.json    vocab.json               target_vocab.json
source.spm               target.spm               README.md
```

别人加载它**不需要 `trust_remote_code`**，三行即可用（见第 4 节）。

为什么"零 .py"很关键：仓库里没有自定义代码，加载时所有实现都来自 `transformers` 库本身，
这就是主流 HF 格式（与 `Helsinki-NLP/opus-mt-*` 同类）。

---

## 0.1 核心文件清单与角色（先看这里）

> 这一节回答两件事：**有哪些核心文件**、**它们各自在本文（这条训练→发布→接入链路）中扮演什么角色**。
> 「随 HF 发布?」列表示该文件**是否会被写进上传到 HuggingFace 的仓库**。

```
config.py ─────────────┐
dataset/*.json ────────┼─► train_marian.py ─► data/train/marian_exp/ ──publish──► HF repo
tokenizer/*.model ─────┘        ▲                         ▲
                                │                         └─(可选) model/* : B 路径自定义架构
                        demo_inference.py ◄── 第三人加载 HF repo / 本地目录
```

### A. 训练 / 发布链路（本文主线）

| 文件                                                                                                             | 在本文中的核心角色                                                                                                                                                          | 关联章节      | 随 HF 发布? |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- | -------- |
| [`train_marian.py`](train_marian.py)                                                                           | **主线入口**：构建分词器与 `MarianMTModel`、数据预处理、`Seq2SeqTrainer` 训练、内存回收、**默认自动续训**、保存标准产物、上传 Hub、上传后自检                                                                      | §2、§3     | 否（是执行脚本） |
| [`demo_inference.py`](demo_inference.py)                                                                       | **第三人接入 Demo**：标准加载（默认不带 `trust_remote_code`）+ 单句/批量/文件/交互式翻译                                                                                                      | §4        | 否        |
| [`config.py`](config.py)                                                                                       | 全局超参与路径（`d_model=512`、`n_heads=8`、`n_layers=6`、`d_ff=2048`、`dropout=0.1`、`max_len=60`、`beam_size=3`、`dataset/*`、`translate_model_path`）；被 `train_marian.py` 读取为默认值 | §1.4、§2.5 | 否        |
| [`dataset/train.json`](dataset/train.json) / [`dev.json`](dataset/dev.json) / [`test.json`](dataset/test.json) | **平行语料**：`[[英文, 中文], ...]`，训练/验证/测试数据源                                                                                                                             | §2.1      | 否        |
| [`tools/tokenizer_utils.py`](tools/tokenizer_utils.py)                                                         | 分词器工具：`english_tokenizer_load()` / `chinese_tokenizer_load()`（原生 spm 直读）+ `hf_tokenizer_load()`（构建 HF `TransformerTokenizer`，B 路径用）                                | §2.3      | 否        |

### B. 分词与词表资产（发布时会写进仓库）

| 文件                                                                               | 在本文中的核心角色                                                                                           | 关联章节      | 随 HF 发布?                |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------- | ----------------------- |
| [`tokenizer/eng.model`](tokenizer/eng.model)                                     | **源端** SentencePiece 模型（英文），编码源句的唯一依据                                                               | §1.1、§2.3 | ✅（发布为`source.spm`）      |
| [`tokenizer/chn.model`](tokenizer/chn.model)                                     | **目标端** SentencePiece 模型（中文），解码译文的唯一依据                                                              | §1.1、§3.2 | ✅（发布为`target.spm`）      |
| [`tokenizer/vocab.json`](tokenizer/vocab.json)                                   | 源端 HF 词表`token -> id`（由 `eng.model` 生成，供 `MarianTokenizer` 映射 id）                                   | §2.3、§3.2 | ✅                       |
| [`tokenizer/target_vocab.json`](tokenizer/target_vocab.json)                     | 目标端 HF 词表`token -> id`（由 `chn.model` 生成）                                                            | §3.2      | ✅                       |
| [`tokenizer/tokenization_transformer.py`](tokenizer/tokenization_transformer.py) | **B 路径**自定义分词器 `TransformerTokenizer(MarianTokenizer)`：源端补 BOS、目标端仅 EOS；并修复上游 `get_tgt_vocab()` bug | §1.7、§3.6 | B 路径才发布（本文 A 主线**不**发布） |

### C. 模型实现

| 文件                                                                                          | 在本文中的核心角色                                                                                                                                                                 | 关联章节 | 随 HF 发布? |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | -------- |
| [`model/configuration_transformer.py`](model/configuration_transformer.py)                  | **B 路径**配置 `TransformerConfig(PretrainedConfig)`（`model_type="transformer_custom"`）                                                                                       | §1.7 | B 路径才发布  |
| [`model/modeling_transformer.py`](model/modeling_transformer.py)                            | **B 路径**模型 `TransformerForConditionalGeneration`（`PreTrainedModel`+`GenerationMixin`）：`src_embed/tgt_embed/encoder/decoder/generator`，参数名与 `tf_model.py` 对齐，可直接载历史 `.pth` | §1.7 | B 路径才发布  |
| [`model/tf_model.py`](model/tf_model.py)                                                    | **历史经典实现**（`make_model`）：本文把它作为 **B 路径权重键名的参照**（新模型与其逐键一致，所以旧 checkpoint 可直接加载）                                                                                           | §1.7 | 否        |
| [`model/factory.py`](model/factory.py)                                                      | 工程内辅助：由`config.py` 构造 `TransformerConfig`、`load_legacy_state_dict()` 加载历史 `.pth`                                                                                          | §1.7 | 否        |
| [`model/__init__.py`](model/__init__.py) / [`tokenizer/__init__.py`](tokenizer/__init__.py) | **B 路径发布机制**：注册 `AutoConfig/AutoModelForSeq2SeqLM/AutoTokenizer` 并调用 `register_for_auto_class` → `save_pretrained` 才会把 `*.py` 写入仓库（`auto_map`+`trust_remote_code`）        | §3.6 | B 路径才发布  |
| [`model/train_utils.py`](model/train_utils.py)                                              | 历史`NoamOpt` / `MultiGPULossCompute`；A 主线改用 HF `Seq2SeqTrainer`，此文件仅为历史参考                                                                                                  | §2.5 | 否        |

### D. 输出：本地训练产物与 HF 仓库

| 文件 / 目录                               | 在本文中的核心角色                                                             | 关联章节      | 随 HF 发布?                 |
| ------------------------------------- | --------------------------------------------------------------------- | --------- | ------------------------ |
| `data/train/marian_exp/checkpoint-*/` | **续训凭据**：`trainer_state.json`（epoch/步数/最优 BLEU）+ 优化器/调度器状态；默认自动续训据此恢复 | §2.8      | 否（被`ignore_patterns` 忽略） |
| `data/train/marian_exp/config.json` 等 | **发布源目录**：`save_pretrained` 产出的标准文件集合                                 | §3.3      | ✅                        |
| `config.json`                         | 结构配置（`model_type=marian`、`share_encoder_decoder_embeddings=false`…）   | §1.4、§3.2 | ✅                        |
| `model.safetensors`                   | 权重（HF 首选安全格式，93.8M 参数）                                                | §3.2      | ✅                        |
| `generation_config.json`              | 生成默认值（`max_length=60`、`num_beams=3`…）                                 | §1.5、§3.3 | ✅                        |
| `tokenizer_config.json`               | 分词器配置（`MarianTokenizer`、`separate_vocabs=True`）                       | §3.2      | ✅                        |
| `README.md`（模型卡）                      | HF 页面元信息 + 用法（front matter：`library_name/pipeline_tag/language/tags`） | §3.3      | ✅                        |
| `HF_MARIAN_GUIDE.md`                  | **本文**：把上述链路（流转/训练/发布/接入）讲清楚                                          | 全文        | 否                        |

### E. 历史 / 已废弃 / 可选脚本（非本文主线）

| 文件                                                                                    | 角色                                                     |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| [`hf_template/`](hf_template)                                                         | 早期"自定义架构模板"（B 路径雏形）；现仅保留 import 别名，**不再作为发布方式**        |
| [`export_hf_repo.py`](export_hf_repo.py)                                              | 旧导出脚本（曾把模板复制进产物）；已被`train_marian.py --publish-only` 取代 |
| [`train_main.py`](train_main.py) / [`translate_main.py`](translate_main.py)           | 历史自研训练/翻译脚本（基于`model/tf_model.py`）                     |
| [`export_onnx*.py`](export_onnx_hf.py) / [`translate_onnx*.py`](translate_onnx_hf.py) | 可选 ONNX 推理分支（与本文 HF 链路独立）                              |

---

## 1. Transformer 在本文中的流转

### 1.1 总览数据流

```
英文句子 (str)
   │  MarianTokenizer(source_spm=eng.model)      # 源端：tokens + </s>（不加 <s>）
   ▼
input_ids (B,S)  +  attention_mask (B,S)
   │
   ├─ Encoder ────────────────────────────────────────────────────────────┐
   │   embed_tokens(src) × sqrt(d_model)          # scale_embedding=True
   │         + 固定 sin-cos 位置编码（Marian 内置，无需学习）
   │   × N=6 层 EncoderLayer：
   │        x = x + dropout( SelfAttn( LayerNorm(x) ) )     # 注意：norm 在子层之前
   │        x = x + dropout( FFN(      LayerNorm(x) ) )
   │   final LayerNorm(x)  ───────────────────────────►  memory (B,S,d_model)
   │
   ▼
中文 labels (B,T)  ──shift_right──► decoder_input_ids (B,T)
   │   decoder_input_ids = [decoder_start_token_id(=2, <s>)] + labels[:-1]
   │   labels 中的 -100（padding）在右移时被替换为 pad_token_id(=0)
   │
   ├─ Decoder ────────────────────────────────────────────────────────────┐
   │   embed_tokens(tgt) × sqrt(d_model) + sin-cos 位置编码
   │   × N=6 层 DecoderLayer：
   │        1) 因果自注意力（mask & 下三角）    x = x + dropout(SelfAttn(LN(x)))
   │        2) 交叉注意力（K,V = memory）        x = x + dropout(CrossAttn(LN(x), memory))
   │        3) 前馈                                  x = x + dropout(FFN(LN(x)))
   │   final LayerNorm(x)  ───────────────────────────►  hidden (B,T,d_model)
   │
   ▼
lm_head: Linear(d_model → tgt_vocab_size=32000)        # 与 decoder embed_tokens 权重绑定
   ▼
logits (B,T,32000)
   │
   ▼
Loss = CrossEntropy(logits, labels, ignore_index=-100)
```

### 1.2 Encoder 细节（为什么与"经典论文 Post-LN"不同）

`MarianEncoderLayer` 的写法是 **norm 在子层之前**：

```
x = x + dropout( sublayer( LayerNorm(x) ) )
```

这与原始论文的 Post-LN（`x = LayerNorm(x + sublayer(x))`）不同，但**自洽且稳定**：
每层前归一化，末端再做一次 `final LayerNorm`。这也是 BART/Marian 系列的标准写法。

- **embedding 缩放**：`embed_scale = sqrt(d_model)`，由 `scale_embedding=True` 打开。
- **位置编码**：Marian 固定使用 `MarianSinusoidalPositionalEmbedding`（sin-cos），无需训练参数，
  因此不存在"位置编码权重"需要迁移的问题。
- **注意力掩码**：源端按 `attention_mask` 屏蔽 padding。
- **激活函数**：本文用 `relu`（`MarianConfig.activation_function="relu"`）。

### 1.3 Decoder 细节

- **因果掩码**：自注意力用下三角 + padding mask，保证第 `t` 步只能看到 `≤ t`。
- **交叉注意力**：`Query` 来自 decoder，`Key/Value` 来自 encoder 的 `memory`。
- **右移（shift right）**：训练时模型不直接看"目标整句"，而是：
  - 输入：`[<s>] + label[:-1]`
  - 目标：`label`（就是 `<eos>` 结尾的整句）
    这样第 `t` 个位置预测的是第 `t` 个目标 token（teacher forcing）。

### 1.4 本文的关键配置（实际写入 `config.json`）

| 字段                                                    | 值                   | 含义                           |
| ----------------------------------------------------- | ------------------- | ---------------------------- |
| `model_type`                                          | `marian`            | 内置架构（无需自定义代码）                |
| `architectures`                                       | `["MarianMTModel"]` | `AutoModelForSeq2SeqLM` 直接识别 |
| `vocab_size`                                          | 32000               | **源端**词表（eng spm）            |
| `decoder_vocab_size`                                  | 32000               | **目标端**词表（chn spm）           |
| `share_encoder_decoder_embeddings`                    | **false**           | 源/目标 embedding**各自独立**（关键）   |
| `d_model`                                             | 512                 | 模型维度                         |
| `encoder_layers` / `decoder_layers`                   | 6 / 6               | 层数                           |
| `encoder_attention_heads` / `decoder_attention_heads` | 8 / 8               | 头数                           |
| `encoder_ffn_dim` / `decoder_ffn_dim`                 | 2048 / 2048         | 前馈维度                         |
| `activation_function`                                 | `relu`              | 激活                           |
| `scale_embedding`                                     | `true`              | embedding × √d_model         |
| `max_position_embeddings`                             | 512                 | sin-cos 位置上限                 |
| `pad_token_id` / `bos_token_id` / `eos_token_id`      | 0 / 2 / 3           | 特殊符号                         |
| `decoder_start_token_id`                              | 2                   | 解码起始符（=`<s>`）                |
| `forced_eos_token_id`                                 | `null`              | 不强制把 pad(0) 当 EOS            |
| 参数量                                                   | **93.8M**           | 6 层 512 维                    |

### 1.5 推理时 Transformer 的流转（`generate`）

```
input_ids ─► Encoder ─► memory（只算一次，被缓存复用）
decoder_input_ids = [decoder_start_token_id(=2)]
loop:
    logits = Decoder+lm_head(decoder_input_ids, memory)
    next_id = argmax / beam-search(logits[:, -1])
    append(next_id)
    if next_id == eos_token_id(=3): break
decode(tokens) ─► 中文（用 target spm 反解）
```

`generation_config.json` 里固化了默认值：`max_length=60`、`num_beams=3`、`early_stopping=true`、
`length_penalty=1.0`、`do_sample=false`。因此第三人只写 `generate(**inputs)` 也能得到合理结果。

### 1.6 一个必须知道的坑（本项目曾因此 BLEU=0）

`transformers` v5 的 `MarianConfig` 中，控制"源/目标是否共用一份 embedding"的字段名是：

```python
share_encoder_decoder_embeddings   # 默认 True
```

而旧文档/直觉常用 `share_encoder_decoder`（**v5 已不是有效字段**，会被当作未知 kwargs 静默忽略）。
本文是**两套独立 SentencePiece**（`eng.model` 与 `chn.model`，id 语义不同）：

- 若 `share_encoder_decoder_embeddings=True`：源/目标的 token id 会强行落在**同一张 embedding** 上，
  两套语义互相冲突 → 训练不收敛（loss 卡在 ~7.6、BLEU=0、输出退化成"这一。"）。
- 必须设为 `False`，并显式给出 `decoder_vocab_size`：

```python
MarianConfig(
    vocab_size=32000,                 # 源端
    decoder_vocab_size=32000,         # 目标端
    share_encoder_decoder_embeddings=False,   # ← 关键
    scale_embedding=True,
    activation_function="relu",
    pad_token_id=0, bos_token_id=2, eos_token_id=3, decoder_start_token_id=2,
    forced_eos_token_id=None,
)
```

> 对照：`Helsinki-NLP/opus-mt-*` 之所以能共享 embedding，是因为它们的源/目标使用**同一套联合词表**
> （id 语义一致）。我们是两套独立 spm，所以必须分开。

### 1.7 另一条路径：自定义架构（本仓库内仍保留）

除 Marian 主线外，仓库还有一套"自定义 Transformer"，用于教学/对比：

- [`model/configuration_transformer.py`](model/configuration_transformer.py)：`TransformerConfig(PretrainedConfig)`，
  `model_type="transformer_custom"`。
- [`model/modeling_transformer.py`](model/modeling_transformer.py)：`TransformerForConditionalGeneration`
  （`PreTrainedModel` + `GenerationMixin`），结构与 `model/tf_model.py` 的经典实现参数名一一对应：
  `src_embed / tgt_embed / encoder / decoder / generator`。

它的流转：`src_embed → encoder → memory`；`tgt_embed + memory + mask → decoder → generator.proj → logits`，
注意力用 `-1e9` 做 mask，**不实现 KV Cache**。发布时需要用 `auto_map` + `trust_remote_code`
（因为是库外架构），仓库会多出 3 个 `.py`——这正是本文主线要避免的形态。

**两条路径对照：**

| 维度         | A. MarianMT（本文主线，已上传）         | B. 自定义架构（备选）                                    |
| ---------- | ----------------------------- | ----------------------------------------------- |
| 代码来源       | `transformers` 内置             | 仓库自带`.py`                                       |
| 仓库是否含`.py` | **否**                         | 是（3 个）                                          |
| 加载         | `from_pretrained(repo)`       | `from_pretrained(repo, trust_remote_code=True)` |
| KV Cache   | 支持（标准`generate`）              | 不实现（每步重算）                                       |
| 训练起点       | 从头训练（共享/独立 embedding 与 B 不通用） | 可加载历史`.pth`                                     |

---

### 1.8 KV Cache 现状（A 有 / B 没有）

| 路径                                                 | 是否实现 KV Cache                | 证据 / 机制                                                                                                                                                       |
| -------------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A 主线：内置 `MarianMTModel`**                        | **是**（由 `transformers` 内置实现） | `modeling_marian.py` 中 `past_key_value` 出现 **63** 次；`_supports_default_dynamic_cache() == True`；`forward` 签名含 `past_key_values`/`use_cache`；`generate` 支持增量解码 |
| **B 路径：自定义 `TransformerForConditionalGeneration`** | **否**                        | `forward` 恒返回 `past_key_values=None`；`_supports_default_dynamic_cache() == False`；`prepare_inputs_for_generation` 固定 `use_cache=False`；每步重算整段 self-attention  |

A 路径实测：

```python
model.config.use_cache = True
model._supports_default_dynamic_cache()          # True
# beam=3 下：use_cache=True 与 use_cache=False 输出完全一致
# → KV Cache 只影响速度，不影响数值结果
```

**一个容易踩的坑**：训练时 HF `Trainer` 会把 `model.config.use_cache` 置为 `False`
（训练不需要 cache）；若原样 `save_pretrained`，仓库里就会是 `"use_cache": false`，
**第三人默认 `generate()` 便不走 KV Cache**（结果正确但更慢）。

本文已在两处修正：

1. 训练保存前：`model.config.use_cache = True`（[`train_marian.py`](train_marian.py)）；
2. 发布前兜底：`publish_local_repo` 发现 `config.json` 的 `use_cache` 不为 `true` 时自动改写为 `true` 再上传。

当前已确认：`Hub config.use_cache = True`，`share_encoder_decoder_embeddings = False`。

## 2. 训练数据与核心代码步骤

对应实现：[`train_marian.py`](train_marian.py)。下面按"一步一函数"拆解。

### 2.1 数据格式

`dataset/train.json`、`dev.json`、`test.json` 均为：

```json
[
  ["The cat is sleeping on the sofa.", "猫在沙发上睡觉。"],
  ["I love you.", "我爱你。"]
]
```

即 `[[源句, 目标句], ...]`。

### 2.2 读取与构建 `datasets.Dataset`

```python
def read_pairs(path, limit=0):
    data = json.load(open(path, encoding="utf-8"))
    if limit: data = data[:limit]
    return [{"en": row[0], "zh": row[1]} for row in data]

raw_train = Dataset.from_list(read_pairs(args.train_file, args.max_train_samples))
raw_eval  = Dataset.from_list(read_pairs(args.dev_file,  args.max_eval_samples))
```

### 2.3 分词：`preprocess`（BOS/EOS 约定是核心）

```python
def preprocess(batch):
    # 源端：tokens + </s>（MarianTokenizer 默认只加 EOS）
    model_inputs = tokenizer(batch["en"], max_length=args.max_source_length, truncation=True)
    # 目标端：tokens + </s>，**不含 BOS**；BOS 由模型 shift_right 用 decoder_start_token_id 补
    labels = tokenizer(text_target=batch["zh"], max_length=args.max_target_length, truncation=True)
    model_inputs["labels"] = labels["input_ids"]
    return model_inputs

train_ds = raw_train.map(preprocess, batched=True, remove_columns=raw_train.column_names)
```

要点：

- `tokenizer(text_target=...)` 会切到**目标端模式**（用 `target.spm` + `target_vocab.json`），
  这是 `MarianTokenizer(separate_vocabs=True)` 的官方用法。
- labels **不含 BOS**，与模型内部 `shift_tokens_right` 组合后才等于 `[BOS] + tokens`，
  避免把 BOS 当预测目标（否则第一个位置会学错）。

实测校验（`dataset/dev.json` 前 2000 条）：

| 项目    | p50 | p90 | p99 | max | 截断(>上限)          |
| ----- | --- | --- | --- | --- | ---------------- |
| 源端长度  | 26  | 45  | 67  | 89  | 0 / 2000（上限 128） |
| 目标端长度 | 20  | 36  | 59  | 92  | 19 / 2000（上限 60） |

> 建议续训时用 `--max-target-length 96` 消除那 ~1% 的目标端截断。

### 2.4 动态 padding：`DataCollatorForSeq2Seq`

```python
data_collator = DataCollatorForSeq2Seq(
    tokenizer=tokenizer,
    model=model,
    label_pad_token_id=-100,   # Marian 内置损失是 CrossEntropy(ignore_index=-100)
    pad_to_multiple_of=8,
)
```

- `input_ids/attention_mask` 按 batch 内最长对齐，`labels` 用 **-100** 填充。
- 采用 **-100**（而不是 pad id）是因为 `MarianMTModel.forward` 内部用
  `CrossEntropyLoss()` 的默认 `ignore_index=-100` 忽略 padding。

### 2.5 训练参数与 Trainer

```python
training_args = Seq2SeqTrainingArguments(
    output_dir=args.output_dir,
    num_train_epochs=args.epochs,
    per_device_train_batch_size=args.batch_size,
    gradient_accumulation_steps=args.grad_accum,
    learning_rate=args.lr,                 # 默认 3e-4
    warmup_steps=args.warmup_steps,        # 默认 4000
    lr_scheduler_type="inverse_sqrt",
    eval_strategy="epoch", save_strategy="epoch",
    save_total_limit=1,
    predict_with_generate=True,            # 验证用生成
    generation_max_length=args.max_target_length,
    generation_num_beams=config.beam_size, # 3
    load_best_model_at_end=True,
    metric_for_best_model="bleu", greater_is_better=True,
    label_names=["labels"],
)

trainer = Seq2SeqTrainer(
    model=model, args=training_args,
    train_dataset=train_ds, eval_dataset=eval_ds,
    data_collator=data_collator,
    processing_class=tokenizer,            # v5 用 processing_class 传分词器
    compute_metrics=build_compute_metrics(tokenizer, model.config.pad_token_id),
    callbacks=[MemoryCleanupCallback(...)],
)
trainer.train(resume_from_checkpoint=args.resume_from_checkpoint)
```

### 2.6 指标：sacreBLEU（中文按字切分）

```python
def compute_metrics(eval_pred):
    preds, labels = eval_pred
    preds  = np.where(preds  == -100, pad_id, preds)
    labels = np.where(labels == -100, pad_id, labels)
    hyps = [p.strip() for p in tokenizer.batch_decode(preds,  skip_special_tokens=True)]
    refs = [l.strip() for l in tokenizer.batch_decode(labels, skip_special_tokens=True)]
    return {"bleu": round(float(sacrebleu.corpus_bleu(hyps, [refs], tokenize="zh").score), 4)}
```

实测（3 轮，同一份数据）：

| epoch | eval_bleu | eval_loss |
| ----- | --------- | --------- |
| 1     | 17.12     | 4.92      |
| 2     | 25.94     | 3.87      |
| 3     | **28.38** | 3.56      |

### 2.7 内存回收（MPS 上必须）

- `import torch` **之前**设置：`PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0`、
  `PYTORCH_MPS_LOW_WATERMARK_RATIO=0.0`、`PYTORCH_ENABLE_MPS_FALLBACK=1`。
- `MemoryCleanupCallback`：每个 step（默认 50）/eval/save/epoch 结束时执行
  `gc.collect()` + `torch.mps.empty_cache()`/`torch.cuda.empty_cache()`。
- `dataloader_pin_memory`：MPS/CPU 默认关闭（`--dataloader-pin-memory auto|on|off`）。
- 可选：`--print-memory` 打印 RSS/设备已分配；`--mps-memory-fraction 0.8` 限制上限。

### 2.8 默认自动续训（`--additional-epochs`）

`train_marian.py` **默认会接着上次产物继续训练**：

1. 在 `--output-dir` 找 step 最大的 `checkpoint-*` → `resume_from_checkpoint`（恢复模型+优化器+调度器+步数）；
2. 若无 checkpoint，但有 `model.safetensors`+`config.json` → 自动 `init_from`（仅权重）；
3. 都没有 → 从头训练。

```bash
# 再练 12 轮（总 = 上次 3 + 12 = 15），并继续用同一目录
python train_marian.py --output-dir data/train/marian_exp --additional-epochs 12 \
  --lr 3e-4 --warmup-steps 2000 --max-target-length 96 \
  --eval-strategy epoch --save-strategy epoch \
  --push-to-hub --hub-model-id chou-lucas/transformer-en-zh

# 关闭自动续训，从头训练
python train_marian.py --output-dir data/train/marian_exp --no-auto-resume
```

### 2.9 一次完整训练

```bash
python train_marian.py --output-dir data/train/marian_exp --epochs 3 \
  --batch-size 16 --lr 3e-4 --warmup-steps 4000 --lr-scheduler-type inverse_sqrt \
  --eval-strategy epoch --save-strategy epoch --save-total-limit 1 \
  --max-source-length 128 --max-target-length 60 \
  --memory-cleanup-steps 50 --dataloader-pin-memory off \
  --push-to-hub --hub-model-id chou-lucas/transformer-en-zh
```

---

## 3. 上传 HuggingFace：标准 / 格式 / 要求 + 上传代码

### 3.1 "标准 HF 仓库"是什么

一个"标准"的 HF 模型仓库，本质要求是：**加载者只需要仓库文件 + `transformers` 库本身**。

- 若架构**内置**（`marian`/`bart`/`t5`…）：仓库**不需要 `.py`**，`AutoModel.from_pretrained(repo)` 直接可用。
- 若架构**库外自定义**：必须走 `auto_map` + `trust_remote_code=True`，仓库需携带 `.py`（本文 B 路径）。

本文主线选择内置 `marian`，因此产出**零 `.py`**。

### 3.2 标准文件清单与含义

| 文件                                 | 作用                                                                                    | 由谁生成                                                |
| ---------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `config.json`                      | 结构配置（`model_type=marian`、`use_cache=true`、`share_encoder_decoder_embeddings=false` 等） | `model.save_pretrained()`                           |
| `model.safetensors`                | 权重（HF 首选安全格式）                                                                         | `model.save_pretrained(safe_serialization=True)`    |
| `generation_config.json`           | 生成默认值（`max_length`/`num_beams`…）                                                      | `model.generation_config` 随 `save_pretrained` 写出    |
| `tokenizer_config.json`            | 分词器配置（`MarianTokenizer`, `separate_vocabs=True`…）                                     | `tokenizer.save_pretrained()`                       |
| `vocab.json` / `target_vocab.json` | 源/目标词表（`token -> id`）                                                                 | `tokenizer.save_vocabulary()`（`save_pretrained` 触发） |
| `source.spm` / `target.spm`        | 源/目标 SentencePiece 模型                                                                 | 同上（从本地`eng/chn.model` 复制）                           |
| `README.md`                        | 模型卡（front matter + 用法）                                                                | 脚本生成                                                |
| `.gitattributes`                   | LFS 规则                                                                                | `create_repo`/上传自动                                  |

模型卡的 front matter（决定 HF 页面上的任务类型与元信息）：

```yaml
---
language: [en, zh]
library_name: transformers
pipeline_tag: translation
tags: [translation, marian, encoder-decoder, sentencepiece]
---
```

### 3.3 上传代码讲解

核心就两步：**保存标准产物** → **创建仓库并上传**（[`train_marian.py`](train_marian.py) 中的
`save_standard_repo` / `publish_local_repo`）。

```python
# ① 保存为标准产物（零 .py）
model.generation_config = build_generation_config()      # 固化 max_length/num_beams 等
model.save_pretrained(out_dir, safe_serialization=True)  # config.json + model.safetensors + generation_config.json
tokenizer.save_pretrained(out_dir)                       # tokenizer_config.json + vocab* + spm
write_model_card_from_dir(out_dir, repo_id)              # README.md

# ② 创建仓库（存在则复用）并整目录上传
from huggingface_hub import HfApi, create_repo
create_repo(repo_id, repo_type="model", exist_ok=True, private=private, token=token)
HfApi(token=token).upload_folder(
    repo_id=repo_id, folder_path=out_dir, token=token,
    commit_message="Upload standard HF MarianMT (en-zh, no custom code)",
    ignore_patterns=["checkpoint-*", "*.pth", "optimizer.pt", "scheduler.pt",
                     "training_args.bin", "trainer_state.json", ".DS_Store"],
)
```

**为什么这样就是"标准"？** 因为保存动作全部交给 HF 官方 API：
`save_pretrained` 负责 `config.json`/权重/生成配置，`save_pretrained`（tokenizer）负责词表与 spm，
不需要手写任何 json，也不需要"模板拷贝"。

### 3.4 迁移/清理：删除旧的自定义代码

如果同名仓库以前是"自定义架构"（带 `.py`），上传前应删除这些遗留文件，保证最终仓库零 `.py`：

```python
existing = {s.rfilename for s in api.model_info(repo_id, token=token).siblings}
for name in ["modeling_transformer.py", "configuration_transformer.py", "tokenization_transformer.py"]:
    if name in existing:
        api.delete_file(path_in_repo=name, repo_id=repo_id, token=token,
                        commit_message=f"Remove {name} (migrate to built-in Marian)")
```

### 3.5 上传后自检（"模拟第三人"）

上传完成后脚本会立刻做一次校验：**从 Hub 下载 → 不带 `trust_remote_code` 加载 → 生成**。

```python
tok = AutoTokenizer.from_pretrained(repo_id)
model = AutoModelForSeq2SeqLM.from_pretrained(repo_id).eval()
out = model.generate(**tok(["..."], return_tensors="pt"), max_length=60, num_beams=3)
```

实测输出（revision `c1b688f9…`）：

```
"The government has implemented various policies to improve the living standards of its citizens."
  -> 政府实施了各种政策改善公民的生活水平。
"I love you." -> 我喜欢你。
```

### 3.6 如果要发布"自定义架构"（对比）

自定义架构**必须**带代码，仓库需要：

```json
// config.json
{
  "model_type": "transformer_custom",
  "auto_map": {
    "AutoConfig": "configuration_transformer.TransformerConfig",
    "AutoModelForSeq2SeqLM": "modeling_transformer.TransformerForConditionalGeneration"
  }
}
```

并调用 `register_for_auto_class(...)`，`save_pretrained()` 才会自动把 `*.py` 写入仓库。
加载方必须写 `trust_remote_code=True`。**这就是"带 .py"与"零 .py"的根本区别。**

### 3.7 发布命令

```bash
# 只导出到本地目录
python train_marian.py --publish-only data/train/marian_exp --hub-model-id chou-lucas/transformer-en-zh
# 训练 + 自动发布
python train_marian.py --output-dir data/train/marian_exp --additional-epochs 12 \
  --push-to-hub --hub-model-id chou-lucas/transformer-en-zh
```

---

## 4. 第三人使用接入

### 4.1 安装

```bash
pip install "transformers>=5" torch sentencepiece sacremoses safetensors
```

### 4.2 最小可用代码（不需要 `trust_remote_code`）

```python
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM

repo = "chou-lucas/transformer-en-zh"
tokenizer = AutoTokenizer.from_pretrained(repo)
model = AutoModelForSeq2SeqLM.from_pretrained(repo)

inputs = tokenizer(["The cat is sleeping on the sofa."], return_tensors="pt", padding=True)
out = model.generate(**inputs, max_length=60, num_beams=3)
print(tokenizer.batch_decode(out, skip_special_tokens=True))
```

### 4.3 批量与常用参数

```python
sents = ["I love you.", "This is a test sentence."]
inputs = tokenizer(sents, return_tensors="pt", padding=True, truncation=True).to("mps")
out = model.generate(**inputs, max_length=96, num_beams=3, early_stopping=True)
print(tokenizer.batch_decode(out, skip_special_tokens=True))
```

- `device`：`cpu` / `mps`(Apple) / `cuda`。
- `num_beams=1` 为贪心（更快）；`3~5` 质量更好。
- 长句可调大 `max_length`；超长建议先分句。

### 4.4 用本地目录 / 离线

```python
tokenizer = AutoTokenizer.from_pretrained("./marian_exp")
model = AutoModelForSeq2SeqLM.from_pretrained("./marian_exp")
```

### 4.5 常见问题

> **KV Cache（增量解码）**：本文仓库的 `config.json` 已设 `use_cache=true`，
> `generate()` 会复用 `past_key_values`，长句更快；若为 `false` 则每步重算整段 attention。
> 详见 [§1.8](#18-kv-cache-现状a-有--b-没有)。

| 现象                      | 原因 / 解决                                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------------------------- |
| 提示需要`trust_remote_code` | 你加载的是"自定义架构"仓库（带`.py`）。本文主线仓库是内置 `marian`，正常不应出现；若确为自定义架构，显式加 `trust_remote_code=True`。                 |
| 输出乱码/退化                 | 大概率是旧版共享 embedding 的坏权重。确认加载的 revision 是修复后的（`config.json` 里 `share_encoder_decoder_embeddings=false`）。 |
| 短句/习语翻译偏弱               | 3 轮只是可用基线；继续用`--additional-epochs` 续训到 15~30 轮会明显改善。                                                    |
| 报缺少`sacremoses`         | `pip install sacremoses`（MarianTokenizer 的规范化可选依赖）。                                                     |

---

## 附录 A：文件速查（完整版）

> 「角色」与「随 HF 发布?」的完整说明见 [§0.1 核心文件清单与角色](#01-核心文件清单与角色先看这里)。

**A. 训练 / 发布链路**

| 文件                                                     | 职责                                                 |
| ------------------------------------------------------ | -------------------------------------------------- |
| [`train_marian.py`](train_marian.py)                   | 主线：内置`MarianMTModel` 训练 + 自动续训 + 零 `.py` 发布 + 上传自检 |
| [`demo_inference.py`](demo_inference.py)               | 第三人接入 Demo（命令行 / 文件 / 交互 / 本地目录）                   |
| [`config.py`](config.py)                               | 全局超参与路径                                            |
| `dataset/{train,dev,test}.json`                        | 平行语料`[[en,zh],...]`                                |
| [`tools/tokenizer_utils.py`](tools/tokenizer_utils.py) | spm 直读 +`hf_tokenizer_load()`                      |

**B. 分词与词表资产**

| 文件                                                                               | 职责                                                  |
| -------------------------------------------------------------------------------- | --------------------------------------------------- |
| `tokenizer/eng.model` / `chn.model`                                              | 源 / 目标 SentencePiece 模型                             |
| `tokenizer/vocab.json` / `target_vocab.json`                                     | 源 / 目标 HF 词表（`token -> id`）                         |
| [`tokenizer/tokenization_transformer.py`](tokenizer/tokenization_transformer.py) | B 路径自定义分词器（源端补 BOS）                                 |
| [`tokenizer/__init__.py`](tokenizer/__init__.py)                                 | B 路径：`AutoTokenizer` 注册 + `register_for_auto_class` |

**C. 模型实现**

| 文件                                                                         | 职责                                          |
| -------------------------------------------------------------------------- | ------------------------------------------- |
| [`model/configuration_transformer.py`](model/configuration_transformer.py) | B 路径配置`TransformerConfig`                   |
| [`model/modeling_transformer.py`](model/modeling_transformer.py)           | B 路径模型`TransformerForConditionalGeneration` |
| [`model/__init__.py`](model/__init__.py)                                   | B 路径：Auto 注册 +`register_for_auto_class`     |
| [`model/factory.py`](model/factory.py)                                     | 由`config.py` 构造配置、加载历史 `.pth`               |
| [`model/tf_model.py`](model/tf_model.py)                                   | 历史经典实现（键名参照）                                |
| [`model/train_utils.py`](model/train_utils.py)                             | 历史`NoamOpt` / `MultiGPULossCompute`         |

**D. 输出**

| 路径                                           | 职责                                                                                                                                                                        |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `data/train/marian_exp/checkpoint-*/`        | 续训凭据（trainer_state/优化器/调度器）                                                                                                                                               |
| `data/train/marian_exp/`                     | 发布源目录（config/safetensors/tokenizer/README）                                                                                                                                |
| HF 仓库                                        | `config.json` / `model.safetensors` / `generation_config.json` / `tokenizer_config.json` / `vocab.json` / `target_vocab.json` / `source.spm` / `target.spm` / `README.md` |
| [`HF_MARIAN_GUIDE.md`](HF_MARIAN_训练与接入文档.md) | 本文                                                                                                                                                                        |

**E. 历史 / 已废弃 / 可选**

| 文件                                                                          | 职责                           |
| --------------------------------------------------------------------------- | ---------------------------- |
| `hf_template/`                                                              | 早期自定义架构模板（已废弃）               |
| [`export_hf_repo.py`](export_hf_repo.py)                                    | 旧导出脚本（已被`--publish-only` 取代） |
| [`train_main.py`](train_main.py) / [`translate_main.py`](translate_main.py) | 历史训练/翻译脚本                    |
| [`export_onnx_hf.py`](export_onnx_hf.py) 等 ONNX 脚本                          | 可选 ONNX 推理分支                 |

## 附录 B：关键命令速查

```bash
# 训练（首次 3 轮）
python train_marian.py --output-dir data/train/marian_exp --epochs 3 --batch-size 16 \
  --lr 3e-4 --warmup-steps 4000 --lr-scheduler-type inverse_sqrt \
  --eval-strategy epoch --save-strategy epoch --save-total-limit 1 \
  --push-to-hub --hub-model-id chou-lucas/transformer-en-zh

# 继续训练（自动找到最新 checkpoint，再练 12 轮）
python train_marian.py --additional-epochs 12 --max-target-length 96 \
  --lr 3e-4 --warmup-steps 2000 --push-to-hub --hub-model-id chou-lucas/transformer-en-zh

# 仅发布已有目录
python train_marian.py --publish-only data/train/marian_exp --hub-model-id chou-lucas/transformer-en-zh

# 推理 Demo
python demo_inference.py --text "I love you."
```

## 附录 C：术语

- **零 .py 仓库**：仓库中不含自定义模型代码，全部实现来自 `transformers`，加载无需 `trust_remote_code`。
- **auto_map**：`config.json` 里声明"自定义类 → 仓库文件"的映射，配合 `trust_remote_code` 使用。
- **separate_vocabs**：源/目标各一套词表与 spm（`MarianTokenizer` 的 `separate_vocabs=True`）。
- **shift right**：把 labels 右移得到 decoder 输入，首位填 `decoder_start_token_id`。
- **teacher forcing**：训练时用真实目标前缀作为 decoder 输入。
