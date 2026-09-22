# Hugging Face Hub + Transformers 使用文档

> 上次我们自己[从训练、推理来实现英翻中小demo](./基于transformer的大模型训练与推理实践.md)，知道整体的一个流程。
> 
> 那么现在我们看下网上开源模型的主流接入流程。
> 
> **本文所有“下载文件内容 / 参数量 / 张量形状 / token id”均来自本机真实缓存与实测运行结果**，
> 缓存的模型为 `Helsinki-NLP/opus-mt-en-zh`，`refs/main` 指向 commit
> `408d9bc410a388e1d9aef112a2daba955b945255`

## 一、基础概念简介

### 1.1 Hugging Face Hub 是什么

Hugging Face Hub（[https://huggingface.co](https://huggingface.co)）是一个开放的「模型 / 数据集 / 应用」托管平台，
可理解为 **AI 领域的 GitHub**：

- **模型仓库（Model Repo）**：每个模型有唯一 `model_id`，形如 `组织名/模型名`，
  本项目为 `Helsinki-NLP/opus-mt-en-zh`。
- **版本管理**：以 git 仓库托管，`refs/main` 指向当前主版本 commit（本项目实测为
  `408d9bc4...`），另存有 `refs/pr/26 → 68e1208c...`。
- **模型卡片（Model Card）**：仓库中的 `README.md`（注意：本项目缓存快照中**并未下载**它，见第三节）。
- **权重与配置文件托管**：仓库中包含权重、配置、分词器文件（详见第三、四节）。

访问入口：

| 入口                  | 说明                                                        |
| ------------------- | --------------------------------------------------------- |
| 网页                  | [https://huggingface.co](https://huggingface.co) 浏览、搜索、下载 |
| `huggingface_hub` 库 | 底层 SDK，负责下载、缓存、上传                                         |
| `transformers` 库    | 上层 API，封装「下载 + 构建对象 + 推理」                                 |



> 国内网络不便时可设镜像（本项目第 3 行即此用途，且必须在 `import transformers` 之前）：
> 
> ```python
> os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
> ```

 

### 1.2 transformers 是什么

`transformers` 是 Hugging Face 官方模型库，核心是 **用统一 API 使用海量预训练模型**：

- **Auto 工厂类**：`AutoTokenizer`、`AutoModelForSeq2SeqLM` 等，按配置自动选实现类。
- **`from_pretrained` / `save_pretrained`**：统一加载权重接口，与 Hub 无缝衔接。
- **`generate`**：统一文本生成接口。



### 1.3 两者的关系

```text
你的代码  ──▶  transformers（用）  ──▶  huggingface_hub（取）  ──▶  HF Hub（存）
                                    └──▶  本地缓存 ~/.cache/huggingface/hub（复用）
```



---

## 二、从「接入」到「使用」的流程

步骤 0：环境准备

```bash
pip install torch transformers
pip install huggingface_hub   # 可选，transformers 通常已依赖
```

步骤 1：配置镜像

```python
import os
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
```

步骤 2：确定 model_id

```python
model_id = "Helsinki-NLP/opus-mt-en-zh"
```

判定依据：该仓库 `config.json` 的 `model_type` 为 `marian`、`architectures` 为 `["MarianMTModel"]`，
`tokenizer_config.json` 声明 `source_lang=eng`、`target_lang=zho`，即英→中翻译。

步骤 3：加载分词器与模型

```python
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForSeq2SeqLM.from_pretrained(model_id).to(device)
```

实测结果：`type(tokenizer).__name__ == "MarianTokenizer"`，
`type(config).__name__ == "MarianConfig"`，模型类为 `MarianMTModel`。

步骤 4：推理

```python
model_inputs = tokenizer([text], return_tensors="pt").to(device)
generated_ids = model.generate(**model_inputs, max_new_tokens=512)
translation = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
```

步骤 5：交互式使用

控制台循环读取英文 → `translate()` → 打印中文，支持空行跳过、`exit/quit/q` 退出、EOF/Ctrl+C 优雅退出。

---

## 三、下载了哪些文件（本机实测）

首次调用 `from_pretrained(model_id)` 时，`huggingface_hub` 把仓库文件下载到本地缓存。

### 3.1 实测文件清单与真实大小

`refs/main` = `408d9bc410a388e1d9aef112a2daba955b945255`，该快照实际只有 **7 个文件**：

| 文件                       | 实际大小                          | 说明                                              |
| ------------------------ | -----------------------------:| ----------------------------------------------- |
| `pytorch_model.bin`      | **312,087,009 B ≈ 297.6 MiB** | 权重（实测 258 个张量，共 144,569,321 参数）                 |
| `vocab.json`             | **1,617,791 B**               | 模型词表，实测**65001** 条，id 范围 0–65000                |
| `source.spm`             | **806,435 B**                 | 源语言(英) SentencePiece 模型，实测 piece_size=32000     |
| `target.spm`             | **804,600 B**                 | 目标语言(中) SentencePiece 模型，实测 piece_size=32000    |
| `config.json`            | **1,403 B**                   | 模型结构配置                                          |
| `generation_config.json` | **293 B**                     | 生成默认参数                                          |
| `tokenizer_config.json`  | **44 B**                      | 仅`{"target_lang": "zho", "source_lang": "eng"}` |

> 与常见模板不同：该快照**没有** `README.md`、`.gitattributes`、`special_tokens_map.json`、
> `merges.txt`、`tokenizer.json`。另外 `refs/pr/26 → 68e1208c...` 快照里只有一个
> `model.safetensors`（同一模型的不同格式 / 版本）。

### 3.2 本地缓存目录结构

```text
~/.cache/huggingface/hub/
└── models--Helsinki-NLP--opus-mt-en-zh/
    ├── blobs/                                  # 真实内容，按 sha256 命名，去重
    │   ├── 69a1d6ec...ffdca3f5  (312,087,009 B) ── pytorch_model.bin
    │   ├── 423d3deb...94bf73e4  (1,617,791 B)   ── vocab.json
    │   ├── 3f695c68...865af02c  (806,435 B)     ── source.spm
    │   ├── e19744e1...e96771cd  (804,600 B)     ── target.spm
    │   ├── e2c7882d...fd99a3c07  (1,403 B)       ── config.json
    │   ├── f07859fc...62eebc851  (293 B)         ── generation_config.json
    │   ├── af90e7a4...cef85740e  (44 B)          ── tokenizer_config.json
    │   └── 2faa8895...62314f2     (312,062,580 B) ── pr/26 的 model.safetensors
    ├── refs/
    │   ├── main      → 408d9bc410a388e1d9aef112a2daba955b945255
    │   └── refs/pr/26 → 68e1208cb5c05563fa24b83785b660c65ba7b047
    └── snapshots/
        ├── 408d9bc4.../   # main：上面 7 个文件皆为指向 blobs 的符号链接
        └── 68e1208c.../   # pr/26：仅 model.safetensors
```

实测 `snapshots/408d9bc4.../` 内文件均为**软链接**（如 `config.json -> ../../blobs/e2c7882d...`），
即「snapshot 只存版本视图，内容存在 blob 中」，因此同一文件多版本不会重复占盘。

### 3.3 相关环境变量

| 变量                                    | 作用                                |
| ------------------------------------- | --------------------------------- |
| `HF_ENDPOINT`                         | 镜像站地址（本项目`https://hf-mirror.com`） |
| `HF_HOME`                             | Hugging Face 根目录                  |
| `HF_HUB_CACHE` / `TRANSFORMERS_CACHE` | 自定义缓存目录                           |
| `HF_HUB_OFFLINE`                      | 设为`1` 强制离线，仅用本地缓存                 |
| `HF_TOKEN`                            | 访问私有 / 受限仓库所需 token               |

---

## 四、这些文件在框架内部如何起作用

理解 `from_pretrained` 的关键：把它拆成两件**互相独立**的事。

1. **读配置 → 造「空壳」**：按 `config.json` 决定「用哪个类」，并按超参数用 `nn.Module`
   搭好网络骨架，**此时参数是随机初始化的**。
2. **读权重 → 填充数值**：把权重张量按参数名逐个塞进骨架。
3. **分词器**：读 `tokenizer_config.json` 选类，再用 `vocab.json` + `*.spm` 建立文本↔id 规则。

### 4.1 `config.json` 实测内容 → 决定类与形状

实测真实内容（节选）：

```json
{
  "activation_function": "swish",
  "architectures": ["MarianMTModel"],
  "bos_token_id": 0,
  "d_model": 512,
  "decoder_attention_heads": 8,
  "decoder_ffn_dim": 2048,
  "decoder_layers": 6,
  "decoder_start_token_id": 65000,
  "encoder_attention_heads": 8,
  "encoder_ffn_dim": 2048,
  "encoder_layers": 6,
  "eos_token_id": 0,
  "is_encoder_decoder": true,
  "max_length": 512,
  "max_position_embeddings": 512,
  "model_type": "marian",
  "num_beams": 4,
  "num_hidden_layers": 6,
  "pad_token_id": 65000,
  "scale_embedding": true,
  "static_position_embeddings": true,
  "vocab_size": 65001
}
```

框架内部的动作（实测验证）：

```text
AutoConfig.from_pretrained(model_id)  →  MarianConfig   (model_type = "marian")
MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING_NAMES["marian"] →  "MarianMTModel"   ← 实测
MarianMTModel(config)                 →  612维空间、6层编解码的网络骨架（随机权重）
```

> 实测对照：`AutoModel`（基类）用的 `MODEL_MAPPING_NAMES["marian"]` 是 `MarianModel`；
> 而 `AutoModelForSeq2SeqLM` 查的是 `MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING_NAMES`，
> 得到 `MarianMTModel`（带 `final_logits_bias` 的翻译头）。这正是“任务决定用哪个 Auto 类”的底层依据。

这些超参数**直接决定张量形状**，与第 4.2 节实测权重一一对应：

| config 字段                             | 值     | 对应实测权重形状                                                |
| ------------------------------------- | -----:| ------------------------------------------------------- |
| `vocab_size`                          | 65001 | `model.shared.weight` → **(65001, 512)**                |
| `d_model`                             | 512   | 各投影矩阵 →**(512, 512)**                                   |
| `encoder_ffn_dim` / `decoder_ffn_dim` | 2048  | `model.decoder.layers.5.fc2.weight` → **(512, 2048)**   |
| `max_position_embeddings`             | 512   | `model.encoder.embed_positions.weight` → **(512, 512)** |
| `pad_token_id`                        | 65000 | `final_logits_bias` → **(1, 65001)** 中的 pad 维度          |

### 4.2 `pytorch_model.bin` 实测结构 → 填充数值

实测 `torch.load(..., map_location="cpu", weights_only=True)` 结果：

```text
state_dict 张量数量 : 258
参数量合计          : 144,569,321  (≈144.6M)
含 "encoder" 的张量 : 158 个
含 "decoder" 的张量 : 158 个

前 6 个参数名与形状：
    final_logits_bias                              (1, 65001)
    model.shared.weight                            (65001, 512)
    model.encoder.embed_tokens.weight              (65001, 512)
    model.encoder.embed_positions.weight           (512, 512)
    model.encoder.layers.0.self_attn.k_proj.weight (512, 512)
    model.encoder.layers.0.self_attn.k_proj.bias   (512,)

后 4 个参数名与形状：
    model.decoder.layers.5.fc2.weight              (512, 2048)
    model.decoder.layers.5.fc2.bias                (512,)
    model.decoder.layers.5.final_layer_norm.weight (512,)
    model.decoder.layers.5.final_layer_norm.bias   (512,)
```

框架内部的动作：

```text
torch.load("pytorch_model.bin")  →  state_dict：{ "参数名": 张量, ... }   (实测 258 项)
                 │
                 ▼
model.load_state_dict(state_dict)  按名字把张量复制进 4.1 造好的层
```

- 参数名必须与 4.1 搭出的网络一致；形状必须与 config 推出的形状一致，否则报 `size mismatch`。
  （实测 `model.shared.weight` 的 (65001, 512) 正好等于 `vocab_size × d_model`。）
- 实测可见 `model.shared.weight` 与 `model.encoder.embed_tokens.weight` **同名异键但形状相同**，
  说明 Marian 把源/目标嵌入绑定到同一 `shared` 矩阵（权重共享）。
- 加载完成后 `from_pretrained` 会自动 `model.eval()`，切到推理模式（关闭 Dropout）。
- 本项目 main 快照用 `pytorch_model.bin`；`refs/pr/26` 提供了等价的 `model.safetensors`
  （312,062,580 B），加载更快、更安全，`.bin` 与 `.safetensors` 是二选一的权重载体。

### 4.3 分词器文件如何协作（实测）

`AutoTokenizer.from_pretrained(model_id)` 的装配链与实测结果：

```text
tokenizer_config.json (44 B) = {"target_lang": "zho", "source_lang": "eng"}
        └─ 仅声明语言对；tokenizer 类由 transformers 内部映射得到 "MarianTokenizer"
vocab.json    (65001 条, id 0–65000)  → token 字符串 ↔ id 映射
source.spm    (piece_size = 32000)    → 源语言(英) 子词切分
target.spm    (piece_size = 32000)    → 目标语言(中) 子词切分
```

实测的特殊 token（并非来自 `tokenizer_config.json`，而由 `vocab.json` / MarianTokenizer 约定）：

```text
tokenizer 实际类 : MarianTokenizer
tokenizer vocab_size : 65001
pad = '<pad>' id=65000 | eos = '</s>' id=0 | unk = '<unk>' id=1
```

**关键机制：`.spm` 只负责“切分”，`vocab.json` 负责“映射成模型 id”。** 实测证据：

```text
source.spm 对 "How are you today?" 的切分结果：
    pieces = ['▁How', '▁are', '▁you', '▁today', '?']
    （spm 自身给的 id 是 [447, 32, 25, 914, 26]，这并不是模型使用的 id）

而 tokenizer 对 "How are you?" 真正产出的：
    pieces      = ['▁How', '▁are', '▁you', '?', '</s>']
    input_ids   = [[906, 46, 39, 25, 0]]
    attention_mask = [[1, 1, 1, 1, 1]]
```

即：`source.spm` 先把英文切成子词，再用 `vocab.json` 把子词查成模型 id
（如 `vocab["▁the"]=3`、`vocab["的"]=12`、`vocab["。"]=10`），最后补 `</s>`(id=0)。
`vocab.json` 中同时含中英子词与语言标签（实测 `">>cmn_Hans<<"=5`、`">>cmn_Hant<<"=21`），
这也是同一个模型能处理多语言方向的体现。解码中文时则用反向映射 + `target.spm` 还原文本。

### 4.4 `generation_config.json` 实测内容 → 生成策略

实测真实内容：

```json
{
  "bad_words_ids": [[65000]],
  "bos_token_id": 0,
  "decoder_start_token_id": 65000,
  "eos_token_id": 0,
  "forced_eos_token_id": 0,
  "max_length": 512,
  "num_beams": 4,
  "pad_token_id": 65000,
  "renormalize_logits": true
}
```

它在加载时成为 `model.generation_config`，`generate()` 每次先读这里的默认值，再被传入参数覆盖：

- `num_beams=4`：默认启用 **beam search**（束宽 4）。
- `eos_token_id=0`：实测生成序列末尾的 `0`，即 `</s>`，据此停止。
- `decoder_start_token_id=65000`：实测生成序列**第一个 id 就是 65000**（见 4.6），解码器的起始符。
- `bad_words_ids=[[65000]]`：禁止把 pad 当作正文输出。
- 本项目只覆盖了 `max_new_tokens=512`，因此会出现实测警告：
  `Both max_new_tokens (=64) and max_length (=512) ... max_new_tokens will take precedence`。

### 4.5 一次 `from_pretrained` 的完整内部时序（实测对照）

```text
AutoTokenizer.from_pretrained(model_id)
  1) 解析 model_id → 定位仓库
  2) 查缓存 ~/.cache/huggingface/hub/models--.../snapshots/<commit>/（命中即免下载）
  3) AutoConfig 读 config.json
  4) 选类 → MarianTokenizer（实测）
  5) 读 vocab.json(65001) + source.spm(32000) + target.spm(32000)
  6) 构建并返回 tokenizer

AutoModelForSeq2SeqLM.from_pretrained(model_id).to(device)
  1) config.json → MarianConfig（实测 model_type="marian"）
  2) 查表 MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING_NAMES →
     "MarianMTModel"（实测）
  3) MarianMTModel(config)  ← 按 d_model=512 / 6 层 造骨架（随机权重）
  4) 载入 pytorch_model.bin（实测 258 张量 / 144.6M 参数）
  5) load_state_dict(...)   ← 按名字填充
  6) model.eval()
  7) 返回 model；代码再 .to(device)
```

一句话：**`config` 定形状、权重填数值、tokenizer 定文本↔id、generation_config 定生成策略**。

### 4.6 端到端实测：`How are you?` → `你好吗?`

用本文件同样的逻辑实测：

```text
输入文本        : How are you?
input_ids       : [[906, 46, 39, 25, 0]]        # source.spm 切分 + vocab.json 映射，末尾补 </s>
attention_mask  : [[1, 1, 1, 1, 1]]
generated_ids   : [[65000, 32157, 25, 0]]       # 65000=decoder_start, 32157='▁你好吗', 25='?', 0=</s>
生成 token      : ['▁你好吗', '?']
最终输出        : 你好吗?
```

这串数字完整印证了 4.1–4.4：`65001` 词表、`65000` 作 `decoder_start_token_id` 与 `pad`、
`0` 作 `eos`。**返回值只含目标侧（不含输入 ids）**，因此直接 `batch_decode` 即可。

### 4.7 可复现验证脚本

脚本内容（逐项对应 4.1–4.6）：

```python
import os

# 必须放在 import transformers 之前才生效
os.environ.setdefault("HF_ENDPOINT", "https://hf-mirror.com")

HF_MODEL_ID = "Helsinki-NLP/opus-mt-en-zh"


def hf_verify_config():
    """[4.1] config.json → 决定模型类与张量形状。"""
    from transformers import AutoConfig
    from transformers.models.auto.modeling_auto import (
        MODEL_MAPPING_NAMES,
        MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING_NAMES,
    )

    cfg = AutoConfig.from_pretrained(HF_MODEL_ID)
    print("config 实际类           :", type(cfg).__name__)
    print("model_type              :", cfg.model_type)
    print("architectures           :", cfg.architectures)
    print("d_model                 :", cfg.d_model)
    print("encoder_layers/decoder  :", cfg.encoder_layers, "/", cfg.decoder_layers)
    print("vocab_size              :", cfg.vocab_size)
    print("decoder_start_token_id  :", cfg.decoder_start_token_id)
    print("MODEL_MAPPING_NAMES['marian'] ->",
          MODEL_MAPPING_NAMES.get(cfg.model_type))
    print("MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING['marian'] ->",
          MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING_NAMES.get(cfg.model_type))
    return cfg


def hf_verify_weights():
    """[4.2] pytorch_model.bin → 张量数量 / 参数量 / 关键形状。"""
    import torch
    from huggingface_hub import hf_hub_download

    try:
        weight_path = hf_hub_download(HF_MODEL_ID, "pytorch_model.bin")
        state_dict = torch.load(weight_path, map_location="cpu", weights_only=True)
    except Exception:  # 兼容仓库改用 safetensors 的情况
        from safetensors.torch import load_file
        weight_path = hf_hub_download(HF_MODEL_ID, "model.safetensors")
        state_dict = load_file(weight_path)

    total = sum(v.numel() for v in state_dict.values() if hasattr(v, "numel"))
    keys = list(state_dict.keys())
    print("文件大小     :", os.path.getsize(weight_path), "bytes")
    print("张量数量     :", len(state_dict))
    print("参数量合计   :", f"{total:,}", f"(≈{total / 1e6:.1f}M)")
    print("含 encoder   :", sum(1 for k in keys if "encoder" in k), "个")
    print("含 decoder   :", sum(1 for k in keys if "decoder" in k), "个")
    for k in keys[:6]:
        print("   ", k, tuple(state_dict[k].shape))
    return state_dict


def hf_verify_tokenizer():
    """[4.3] tokenizer_config.json + vocab.json + *.spm → 文本↔id。"""
    import json
    from transformers import AutoTokenizer
    from huggingface_hub import hf_hub_download

    tok = AutoTokenizer.from_pretrained(HF_MODEL_ID)
    print("tokenizer 实际类 :", type(tok).__name__)
    print("tokenizer vocab  :", tok.vocab_size)
    print("特殊 token       : pad=", repr(tok.pad_token), tok.pad_token_id,
          "| eos=", repr(tok.eos_token), tok.eos_token_id,
          "| unk=", repr(tok.unk_token), tok.unk_token_id)

    vocab = json.load(open(hf_hub_download(HF_MODEL_ID, "vocab.json"), encoding="utf-8"))
    print("vocab.json 条目数:", len(vocab), "| id 范围:",
          min(vocab.values()), "->", max(vocab.values()))
    print("vocab 映射示例   :", {p: vocab.get(p) for p in
          ["</s>", "<unk>", "▁the", "的", "。", ">>cmn_Hans<<"]})

    try:
        import sentencepiece as spm
        for name in ("source.spm", "target.spm"):
            sp = spm.SentencePieceProcessor()
            sp.Load(hf_hub_download(HF_MODEL_ID, name))
            print(f"{name}: piece_size={sp.GetPieceSize()}")
    except ImportError:
        print("(未安装 sentencepiece，跳过 .spm 细节)")

    text = "How are you?"
    enc = tok([text], return_tensors="pt")
    print("输入文本         :", text)
    print("source.spm 切分  :", tok.convert_ids_to_tokens(enc["input_ids"][0]))
    print("→ input_ids      :", enc["input_ids"].tolist(),
          "（经 vocab.json 映射，非 spm 自身 id）")
    print("→ attention_mask :", enc["attention_mask"].tolist())
    return tok


def hf_verify_generation_config():
    """[4.4] generation_config.json → generate() 的默认策略。"""
    from transformers import GenerationConfig

    gen_cfg = GenerationConfig.from_pretrained(HF_MODEL_ID)
    print("num_beams              :", gen_cfg.num_beams)
    print("max_length             :", gen_cfg.max_length)
    print("eos_token_id           :", gen_cfg.eos_token_id)
    print("decoder_start_token_id :", gen_cfg.decoder_start_token_id)
    print("bad_words_ids          :", gen_cfg.bad_words_ids)
    return gen_cfg


def hf_verify_end_to_end(device):
    """[4.6] 端到端：编码 → 生成 → 解码。"""
    import torch
    from transformers import AutoTokenizer, AutoModelForSeq2SeqLM

    tok = AutoTokenizer.from_pretrained(HF_MODEL_ID)
    model = AutoModelForSeq2SeqLM.from_pretrained(HF_MODEL_ID).to(device).eval()

    enc = tok(["How are you?"], return_tensors="pt").to(device)
    with torch.no_grad():
        generated_ids = model.generate(**enc, max_new_tokens=64)
    print("input_ids     :", enc["input_ids"].tolist())
    print("generated_ids :", generated_ids.tolist())
    print("最终输出      :",
          tok.batch_decode(generated_ids, skip_special_tokens=True)[0])
    return generated_ids


if __name__ == "__main__":
    hf_verify_config()
    hf_verify_weights()
    hf_verify_tokenizer()
    hf_verify_generation_config()
    hf_verify_end_to_end(device)   # transformer_simple_test.py 中 device 来自 config.device
```

运行后的关键输出（与 4.1–4.6 一致）：

```text
config 实际类 : MarianConfig | model_type : marian | architectures : ['MarianMTModel']
MODEL_FOR_SEQ_TO_SEQ_CAUSAL_LM_MAPPING['marian'] -> MarianMTModel
张量数量 : 258 | 参数量合计 : 144,569,321 (≈144.6M)
tokenizer 实际类 : MarianTokenizer | vocab : 65001
特殊 token : pad= '<pad>' 65000 | eos= '</s>' 0 | unk= '<unk>' 1
vocab.json 条目数: 65001 | id 范围: 0 -> 65000
source.spm: piece_size=32000
target.spm: piece_size=32000
input_ids     : [[906, 46, 39, 25, 0]]
generated_ids : [[65000, 32157, 25, 0]]
最终输出      : 你好吗?
```

---

## 五、本文件代码核心原理

### 5.1 为什么用 `Auto*` 类

`AutoTokenizer` / `AutoModelForSeq2SeqLM` 读取 `config.json` 的 `model_type`，
经各自映射表决定实际类。实测 `marian` → `MarianMTModel`。
换 `t5-small`（`t5`→`T5ForConditionalGeneration`）、`facebook/bart-base` 时**只需改 `model_id`**。

### 5.2 任务与模型类匹配

| 任务类型             | 应使用的 Auto 类                          | 本项目      |
| ---------------- | ------------------------------------ | -------- |
| 翻译 / 摘要（Seq2Seq） | `AutoModelForSeq2SeqLM`              | ✅        |
| 因果语言模型 / Chat    | `AutoModelForCausalLM`               | 原 chat 版 |
| 文本分类             | `AutoModelForSequenceClassification` | —        |
| 编码器特征            | `AutoModel`                          | —        |

### 5.3 编码 → 生成 → 解码（核心三段）

1. **编码**（第 29 行）：`tokenizer([text], return_tensors="pt").to(device)`
   实测产出 `input_ids=[[906,46,39,25,0]]`、`attention_mask=[[1,1,1,1,1]]`。
2. **生成**（第 33–36 行）：`model.generate(**model_inputs, max_new_tokens=512)`
   实测返回 `[[65000, 32157, 25, 0]]`，**仅含目标侧**。
3. **解码**（第 39 行）：`tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]`
   去掉 `65000`/`0` 等特殊 token，实测得 `你好吗?`。

### 5.4 `generate` 内部：Seq2Seq 解码怎么跑起来

```text
input_ids + attention_mask ─▶ Encoder ─▶ encoder_hidden_states
                                              │ (cross-attention K/V)
                                              ▼
decoder_start_token_id(65000) ─▶ Decoder ─▶ logits
                                   │ 逐 token 选择（beam=4）并回灌
                                   ▼
                      命中 eos_token_id(0) 或达 max_new_tokens 停止
                                   ▼
                         generated_ids（实测 [[65000, 32157, 25, 0]]）
```

- `attention_mask` 屏蔽 padding，避免干扰语义表示。
- `decoder_start_token_id` 来自 `generation_config.json`（实测 65000），故返回值不含输入前缀；
  这与 `ForCausalLM`（原 chat 版需切片）不同。
- cross-attention 让解码器每步“看向”英文语义，翻译对齐在此发生。

### 5.5 设备管理与交互循环

- [`get_device()`](common/device.py) 决定 CPU/GPU；`.to(device)` 保证张量与模型同设备。
- 第 43–61 行 `while True` + `input()`：`strip()`、空行跳过、`exit/quit/q` 退出、
  捕获 `EOFError`/`KeyboardInterrupt` 优雅退出。

### 5.6 数据流全景

```text
控制台英文输入
      │ input()
      ▼
   translate(text)
      │ tokenizer([text], return_tensors="pt")  →  source.spm 切分 + vocab.json 映射
      ▼
 input_ids + attention_mask ─.to(device)─▶ model.generate(max_new_tokens=512)
      │                                          │（beam=4, decoder_start=65000, eos=0）
      │                                          ▼
      │                                   generated_ids（仅目标侧）
      │ tokenizer.batch_decode(skip_special_tokens=True)  →  target.spm 还原
      ▼
   中文译文 ─▶ print()
```

实测交互：

```text
Using device: cpu
模型和分词器加载完成！
英译中翻译器已启动，输入英文句子进行翻译（输入 exit / quit / q 退出）：

请输入英文: How are you?
中文翻译: 你好吗?

请输入英文: q
已退出。
```

---

## 六、常见问题（FAQ）

| 现象                                   | 原因与解决                                                                 |
| ------------------------------------ | --------------------------------------------------------------------- |
| 下载超时 / 连接失败                          | 设`HF_ENDPOINT=https://hf-mirror.com`，务必在 `import transformers` 前      |
| 重复下载                                 | 缓存目录被清空，或`TRANSFORMERS_CACHE` 指向了不同路径                                 |
| `max_new_tokens` 与 `max_length` 同设告警 | 来自`generation_config.json` 的 `max_length=512`；`max_new_tokens` 优先，可忽略 |
| 输出含`<pad>`(65000)                    | 解码时忘记`skip_special_tokens=True`                                       |
| 输出含`</s>`(0)                         | 同上；`eos_token_id=0` 会出现在生成序列末尾                                        |
| 显存不足                                 | 改用 CPU、用小模型或更小 batch（本模型实测约 144.6M 参数）                                |
| 想离线运行                                | 设`HF_HUB_OFFLINE=1`，确保本地已有对应 snapshot 缓存                              |
