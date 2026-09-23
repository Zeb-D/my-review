# HuggingFace 风格导出与接入文档（Transformer en → zh）

本文档说明本工程（`transformers_learning/`）如何把 [`train_main.py`](transformers_learning/train_main.py) 训练出来的权重
`data/train/exp/weights/best_bleu_26.30.pth`，导出成 **可以直接上传 HuggingFace Hub、别人 clone 下来就能跑** 的模型仓库，
以及使用方（消费端）如何接入。

## 0. 总览：三个脚本 + 一份模板

| 脚本 / 目录                                                                  | 作用                                                                                                    | 默认产物                |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- | ------------------- |
| [`export_onnx_hf.py`](transformers_learning/export_onnx_hf.py:1004)      | 按 HuggingFace / Optimum 的 seq2seq 规范导出 **只含推理图** 的 ONNX，并附 HF 风格的元数据与分词器文件                            | `<ckpt目录>/onnx_hf/` |
| [`export_hf_repo.py`](transformers_learning/export_hf_repo.py:594)       | 导出 **完整 HF 模型仓库**（safetensors + `auto_map` + 自定义代码 + 分词器 + ONNX + 模型卡 + 自检），并支持 `--upload` 上传 + 上传后校验 | `<ckpt目录>/hf_repo/` |
| [`translate_onnx_hf.py`](transformers_learning/translate_onnx_hf.py:815) | 消费端：纯 onnxruntime 加载 ONNX 目录 / 直接从 Hub 拉取做英译中，可与 PyTorch 逐 token 对照，并支持 `--verify-hub` 下载验证           | —（可当库调用）            |
| [`hf_template/`](transformers_learning/hf_template)                      | 随仓库一起分发的自定义 `config` / `modeling` / `tokenization` 代码                                                 | 被复制进 `hf_repo/`     |

`export_hf_repo.py` 并不重复实现 ONNX 导出，它直接复用
[`export_onnx_hf.py`](transformers_learning/export_onnx_hf.py:387) 里的
[`build_and_load_model()`](transformers_learning/export_onnx_hf.py:301)、
[`HFEncoderModel`](transformers_learning/export_onnx_hf.py:175) / [`HFDecoderModel`](transformers_learning/export_onnx_hf.py:187)、
[`export_models()`](transformers_learning/export_onnx_hf.py:387) 与
[`verify_onnx()`](transformers_learning/export_onnx_hf.py:448)，两者共用同一套图定义。
所以：**先理解 `export_onnx_hf.py`，`export_hf_repo.py` 就是在它外面再包一层 HF 仓库产物。**

本机已真实导出的目录（数字均取自实测）：

| 目录                                | 文件数 | 体积       | 用途                          |
| --------------------------------- | --- | -------- | --------------------------- |
| `data/train/exp/weights/onnx_hf/` | 9   | ≈ 398 MB | 纯 ONNX + 元数据（对齐 optimum 布局） |
| `data/train/exp/weights/hf_repo/` | 16  | ≈ 790 MB | 完整 HF 仓库（可直接 `hf upload`）   |

---

## 1. 使用 HuggingFace 的前提条件

### 1.1 软件依赖

| 依赖              | 版本要求                         | 用途                                                                              | 缺失时的表现                               |
| --------------- | ---------------------------- | ------------------------------------------------------------------------------- | ------------------------------------ |
| Python          | ≥ 3.9（本机 `~/.penv` 为 3.13.3） | 运行导出/推理脚本                                                                       | —                                    |
| torch           | ≥ 2.0（本机 2.14.0）             | 建模型、加载 `.pth`、导出 ONNX                                                           | 直接报错                                 |
| transformers    | ≥ 4.40（本机 5.16.1）            | `AutoTokenizer` / `AutoModelForSeq2SeqLM` / `generate()`；版本号会写进 `config.json`   | `export_hf_repo.py` 的自检会失败           |
| sentencepiece   | 必需                           | 读 `tokenizer/eng.model`、`tokenizer/chn.model` 与仓库里的 `source.spm` / `target.spm` | 读不了分词器；导出时 `--no-tokenizer` 可降级      |
| sacremoses      | 必需（HF 侧）                     | `MarianTokenizer` 家族的分词依赖（自定义分词器继承它）                                            | 加载 tokenizer 时报缺包                    |
| safetensors     | 推荐                           | 生成 HF 首选权重格式 `model.safetensors`                                                | 自动退回 `pytorch_model.bin`（仍是合法 HF 仓库） |
| onnx            | 必需（导出）                       | ONNX proto 读写、`onnx.checker`                                                    | 两个导出后端都跑不起来                          |
| onnxscript      | 推荐                           | 新版 `dynamo(torch.export)` 导出后端需要它                                               | 自动回退到 `legacy(TorchScript)` 后端       |
| onnxruntime     | 必需（校验/推理）                    | 数值对比、动态轴检查、`translate_onnx_hf.py`                                               | 校验被跳过（仅 WARN），消费端跑不了                 |
| huggingface_hub | 仅上传需要                        | `hf upload` / `create_repo`                                                     | 只能手动网页上传                             |

一次性安装：

```bash
pip install "transformers>=4.40" torch sentencepiece sacremoses safetensors \
            onnx onnxscript onnxruntime huggingface_hub
```

> 导出本身只依赖 `torch + onnx(+onnxscript)`；`transformers` 只用于**导出后自检**
> （[`verify_hf_repo()`](transformers_learning/export_hf_repo.py:390) 用 `trust_remote_code=True` 真实加载仓库），
> 以及写入 `transformers_version` 字段。

### 1.2 本工程侧必须具备的文件

| 路径                                                                        | 说明                                                       |
| ------------------------------------------------------------------------- | -------------------------------------------------------- |
| `data/train/exp/weights/best_bleu_26.30.pth`                              | 训练产出的权重（393.9 MB，纯 `state_dict`，262 个张量）                 |
| `tokenizer/eng.model` / `tokenizer/chn.model`                             | 源/目标 SentencePiece 模型，导出时复制为 `source.spm` / `target.spm` |
| `hf_template/{configuration,modeling,tokenization}_transformer_custom.py` | 自定义架构代码模板，导出时原样复制进仓库                                     |
| `dataset/dev.json`                                                        | 自检/演示用句对（`[英文, 中文]` 数组），默认取前 2 句做自检                      |

`--ckpt` 默认取 [`config.translate_model_path`](transformers_learning/config.py:62)；
脚本用 [`resolve_path()`](transformers_learning/export_onnx_hf.py:272) 同时支持「相对当前目录」和「相对脚本目录」两种写法。

### 1.3 为什么使用方必须开 `trust_remote_code=True`

本模型的架构（Post-LN + sin-cos 位置编码 + 两套独立词表）**不是** HF 内置的 BART/Marian 等结构，因此：

* `config.json` 里 `model_type = transformer_custom`，并通过 `auto_map` 指向仓库内的 `.py`：
  
  ```json
  "auto_map": {
    "AutoConfig": "configuration_transformer_custom.TransformerCustomConfig",
    "AutoModelForSeq2SeqLM": "modeling_transformer_custom.TransformerForConditionalGeneration"
  }
  ```
  
  代码随仓库分发（[`configuration_transformer_custom.py`](transformers_learning/hf_template/configuration_transformer_custom.py:26)、
  [`modeling_transformer_custom.py`](transformers_learning/hf_template/modeling_transformer_custom.py:288)）。

* 分词器也必须用仓库里的自定义类（[`TransformerCustomTokenizer`](transformers_learning/hf_template/tokenization_transformer_custom.py:25)），
  它做了两件训练时必需的事：固定 `separate_vocabs=True`（编码用英文 spm、解码用中文 spm）、
  **句首补 BOS**（`[BOS] + pieces + [EOS]`，与训练侧完全一致）。

后果：**漏写 `trust_remote_code=True`** 时，HF 会退回到内置的 `MarianTokenizer`
——它只补 EOS、且用英文 spm 解码，翻译质量会明显下降（模型侧则因 `model_type` 未知而直接加载失败）。

### 1.4 上传（可选）的前提

```bash
huggingface-cli login            # 或 export HF_TOKEN=hf_xxx
# 国内网络可设置镜像：export HF_ENDPOINT=https://hf-mirror.com
```

* 未登录时导出照常完成，只是 `--upload` 会失败并提示手动 `hf upload`。
* 私有仓库/组织仓库需要在 `--repo-id` 里带上命名空间，并保证 token 有**写权限**。

### 1.5 磁盘与时间预算

* 单次导出的磁盘占用：`onnx_hf/` ≈ 398 MB；`hf_repo/` ≈ 790 MB（其中 `model.safetensors` 375.6 MB、
  `encoder_model.onnx` 145.3 MB、`decoder_model.onnx` 232.5 MB）。
* ONNX 的权重与 safetensors 是同一份参数的两种容器，**放在同一个仓库里会有约 2 倍冗余**；
  只想省空间就加 `--no-onnx`（导出纯 HF 仓库）或只跑 `export_onnx_hf.py`。
* 本机实测：导出 + 双向自检不到 2 分钟（dynamo 后端、CPU）。

### 1.6 校验能力的前提（建议开启）

安装了 `onnx` 与 `onnxruntime` 时，导出后会自动做三类校验（[`verify_onnx()`](transformers_learning/export_onnx_hf.py:448)）：

1. `onnx.checker` 结构合法性；
2. onnxruntime 与 PyTorch 的**逐值**对比（`max_abs_diff` / `allclose`）以及 **decoder 每步 argmax 一致率**；
3. 换一组 `batch / sequence_length` 再跑一次，确认动态轴没被写死。

缺 `onnx` 只跳过第 1 项，缺 `onnxruntime` 则第 2、3 项一起跳过（日志 WARN，导出仍会成功）。

---

## 2. export 流程及产生物清单

### 2.1 命令

```bash
cd transformers_learning

# ① 只导 ONNX（HF/optimum 风格目录）
python export_onnx_hf.py                       # → data/train/exp/weights/onnx_hf
python export_onnx_hf.py --output log_probs    # decoder 输出 log_softmax（默认是 HF 约定的 logits）
python export_onnx_hf.py --demo --text "I love you."   # 导出后跑一次真实翻译+对照

# ② 导完整 HF 仓库（内含 ONNX）
python export_hf_repo.py --repo-id your-name/en-zh-base
python export_hf_repo.py --clean --no-verify               # 快速重导，跳过自检
python export_hf_repo.py --no-onnx                         # 只要 PyTorch 部分（省磁盘）
python export_hf_repo.py --repo-id your-name/en-zh-base --upload your-name/en-zh-base   # 导出即上传

# ③ 消费端验证（纯 onnxruntime）
python translate_onnx_hf.py                                              # 交互式，q! 退出
python translate_onnx_hf.py --from-dev 5 --decode beam --no-compare      # 批量，且不需要 torch
python translate_onnx_hf.py --onnx-dir data/train/exp/weights/onnx_hf --show-details
```

常用开关：`--opset`（默认 18）、`--exporter {auto,dynamo,legacy}`、`--patch {auto,always,never}`、
`--output {logits,log_probs}`、`--batch-size/--src-len/--tgt-len`（dummy 输入形状）、
`--no-verify`、`--coreml`（用 CoreML EP 校验/推理，更快但可能引入数值差异）。

### 2.2 流程步骤

以 [`export_hf_repo.py:main()`](transformers_learning/export_hf_repo.py:594) 为例（`export_onnx_hf.py` 是它的子集）：

1. **解析参数、定位目录**：`--ckpt` → 权重路径；`--repo-dir` 默认 `<ckpt目录>/hf_repo`；`--clean` 先删旧产物
   （[`GENERATED_FILES`](transformers_learning/export_hf_repo.py:85)，不会误删目录里的其它文件）。
2. **建模型 + 载权重**（[`build_and_load_model()`](transformers_learning/export_onnx_hf.py:301)）：
   按 [`config.py`](transformers_learning/config.py) 的 `d_model/n_heads/n_layers/d_ff/dropout` 调 `make_model`，
   权重用 `weights_only=True` 加载（失败回退 `False`），自动剥掉 `module.` 前缀，`strict=True` 失败才降级为非严格并打印缺失/多余键；
   随后 `model.eval()` + `requires_grad_(False)`——**只做推理**。
3. **包一层 HF 风格图**：encoder 用 [`HFEncoderModel`](transformers_learning/export_onnx_hf.py:175)
   （`input_ids + attention_mask → last_hidden_state`），decoder 用 [`HFDecoderModel`](transformers_learning/export_onnx_hf.py:187)
   （`input_ids + encoder_hidden_states + encoder_attention_mask → logits`）。
4. **mask 在图内构造**，规则与训练侧 `tools/data_loader.py` 完全一致：
   [`make_src_mask()`](transformers_learning/export_onnx_hf.py:134) 把 HF 语义的 `attention_mask`（真实 token=1）
   变成 `(B,1,S)`；[`make_tgt_mask()`](transformers_learning/export_onnx_hf.py:139) 生成 padding mask & 下三角 `(B,T,T)`。
5. **造 dummy 输入**（[`OnnxSeq2SeqConfig.dummy_inputs()`](transformers_learning/export_onnx_hf.py:256)）：
   id 落在 `[1, vocab)`，并特意把最后一行最后一列设成 padding，用来覆盖 mask 分支。
6. **导出 ONNX**（[`export_onnx()`](transformers_learning/export_onnx_hf.py:343)）：
   优先 `dynamo(torch.export)`，失败自动回退 `legacy(TorchScript)`；opset 18；
   `dynamic_axes` 声明 batch / sequence_length / encoder_sequence_length 为动态维；`external_data=False` 保证单文件。
   模型里两处对导出不友好的写法由 [`patch_model_for_onnx()`](transformers_learning/export_onnx_hf.py:150) 兜底
   （`--patch auto`：**先不打补丁**，只有导出失败才打，并打印补丁前后差异；本次导出 `patch_applied=false`）。
7. **校验**（[`verify_onnx()`](transformers_learning/export_onnx_hf.py:448)）：`onnx.checker` + onnxruntime 逐值对比 +
   argmax 一致率 + 动态轴（detail 见 §2.6）。
8. **写 HF 风格产物 + 分词器资产**（[`write_artifacts()`](transformers_learning/export_onnx_hf.py:690)、
   [`write_tokenizer_assets()`](transformers_learning/export_hf_repo.py:217)）：`config.json`、`generation_config.json`、
   `tokenizer_config.json`、`special_tokens_map.json`、`vocab.json` / `target_vocab.json`、
   `source.spm` / `target.spm`、`README.md`（模型卡，含 YAML front matter）；
   `export_hf_repo.py` 还会把 `model.safetensors` 与 `hf_template/` 的三个 `.py` 复制进来。
9. **自检**（[`verify_hf_repo()`](transformers_learning/export_hf_repo.py:390)）：完全模拟“别人拿到仓库”——
   `AutoTokenizer` / `AutoModelForSeq2SeqLM` + `trust_remote_code=True` 加载，核对**分词 id 与训练侧人工编码是否一致**、
   `generate()`（贪心 + beam）与 ONNX 解码是否**逐 token 一致**、以及 HF 与 ONNX 的 logits 最大绝对误差；
   结果写入 `export_meta.json`，并摘录进模型卡。
10. **（可选）上传**：`--upload <repo-id>` 用 `create_repo(exist_ok=True)` + `upload_folder` 直接推送。

### 2.3 产物清单 A：`hf_repo/`（完整 HF 仓库，16 个文件 ≈ 790 MB）

| 文件                                    | 大小                | 说明                                                                                                                                                                                                                                                                           |
| ------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `config.json`                         | 820 B             | [`PretrainedConfig`](transformers_learning/export_hf_repo.py:122) 风格：`model_type=transformer_custom`、`architectures`、`auto_map`、结构超参、pad/unk/bos/eos=0/1/2/3、`pre_norm=false`、`scale_embedding=true`、`tie_word_embeddings=false`、`output_mode=logits`、`transformers_version` |
| `model.safetensors`                   | 375.6 MB          | PyTorch 权重；键名与 [`model/tf_model.py`](transformers_learning/model/tf_model.py) 完全一致，`from_pretrained` 无需改键                                                                                                                                                                    |
| `modeling_transformer_custom.py`      | 18.9 KB           | `PreTrainedModel` 实现（[`TransformerForConditionalGeneration`](transformers_learning/hf_template/modeling_transformer_custom.py:288)），支持 `generate()`，不实现 KV Cache                                                                                                             |
| `configuration_transformer_custom.py` | 3.3 KB            | [`TransformerCustomConfig`](transformers_learning/hf_template/configuration_transformer_custom.py:26)                                                                                                                                                                        |
| `tokenization_transformer_custom.py`  | 1.6 KB            | [`TransformerCustomTokenizer`](transformers_learning/hf_template/tokenization_transformer_custom.py:25)（源/目标词表分离 + 句首 BOS）                                                                                                                                                   |
| `encoder_model.onnx`                  | 145.3 MB          | `input_ids + attention_mask → last_hidden_state`                                                                                                                                                                                                                             |
| `decoder_model.onnx`                  | 232.5 MB          | `input_ids + encoder_hidden_states + encoder_attention_mask → logits`                                                                                                                                                                                                        |
| `tokenizer_config.json`               | 621 B             | `tokenizer_class=MarianTokenizer` 保底 + `auto_map` 指向自定义类，`separate_vocabs=true`，special token 映射                                                                                                                                                                             |
| `generation_config.json`              | 306 B             | `max_length=60`、`num_beams=3`、`early_stopping=true`、`length_penalty=1.0`、**`use_cache=false`**、`decoder_start_token_id=2`                                                                                                                                                    |
| `special_tokens_map.json`             | 95 B              | `<s> </s> <unk> <pad>`                                                                                                                                                                                                                                                       |
| `vocab.json` / `target_vocab.json`    | 613 KB / 602 KB   | 源/目标词表（`piece → id`，HF 分词器格式）                                                                                                                                                                                                                                                |
| `source.spm` / `target.spm`           | 0.76 MB / 0.75 MB | 英文 / 中文 SentencePiece 模型（由 `tokenizer/eng.model`、`chn.model` 复制）                                                                                                                                                                                                             |
| `README.md`                           | 6.4 KB            | 模型卡：front matter（`library_name: transformers`、`pipeline_tag: translation`）+ 文件清单 + 三种用法 + 限制                                                                                                                                                                                 |
| `export_meta.json`                    | 3.2 KB            | 本工程附加：权重路径、ONNX 校验、HF 自检、生成配置摘要（HF 无此文件）                                                                                                                                                                                                                                     |
| `pytorch_model.bin`                   | —                 | 仅在未安装 `safetensors` 时出现的降级产物（本次未生成）                                                                                                                                                                                                                                          |

### 2.4 产物清单 B：`onnx_hf/`（ONNX 目录，9 个文件 ≈ 398 MB）

`export_onnx_hf.py` 的输出比 `hf_repo/` 少掉「PyTorch 权重 + 自定义代码 + 词表 json」，
`tokenizer_config.json` 也更朴素（`tokenizer_class: SentencePieceTokenizer`，`add_bos_token/add_eos_token=true`）。
它**不是**一个能被 `transformers` 直接加载的完整仓库，而是「ONNX 模型 + 自描述元数据 + 分词器」的最小自包含目录，
面向纯 onnxruntime 消费：

```
/config.json  /generation_config.json  /tokenizer_config.json
/source.spm   /target.spm
/encoder_model.onnx   /decoder_model.onnx
/README.md    /export_meta.json
```

`export_meta.json` 关键字段：`onnx_opset=18`、`exporter=auto`、`output_mode=logits`、
`exported_at_dtype=float32`、`inference_only=true`、`patch_applied=false`、`patch_equivalence_max_abs_diff=0.0`、
`files[]`（每个 ONNX 的输入输出签名）、`verification`（§2.6 的全部校验数据）。

### 2.5 产物清单 C：旧的 `onnx/`（[`export_onnx.py`](transformers_learning/export_onnx.py) 的输出，4 个文件 ≈ 793 MB）

早期脚本的产物，**输入输出名与 HF/optimum 不同**，只被 [`translate_onnx.py`](transformers_learning/translate_onnx.py) 使用：

| 文件                         | 说明                                                   |
| -------------------------- | ---------------------------------------------------- |
| `transformer_full.onnx`    | `src, tgt → log_probs`，整模型一次前向（teacher forcing / 打分） |
| `transformer_encoder.onnx` | `src → memory`                                       |
| `transformer_decoder.onnx` | `tgt, memory, src_mask → log_probs`                  |
| `export_meta.json`         | 校验元信息                                                |

与 HF 版的对应关系：`src ↔ input_ids`、`memory ↔ encoder_hidden_states`、`src_mask ↔ encoder_attention_mask`、
`log_probs ↔ logits`（前者是 log_softmax 后、后者是未归一化）。**新接入请优先用 `hf_repo/` 或 `onnx_hf/`。**

### 2.6 接口约定（ONNX 输入输出）

| 模型                   | 输入                                                                                                                                                                                           | 输出                                                          |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `encoder_model.onnx` | `input_ids` int64 `(batch, sequence_length)`<br>`attention_mask` int64 `(batch, sequence_length)`，真实 token=1                                                                                 | `last_hidden_state` float32 `(batch, sequence_length, 512)` |
| `decoder_model.onnx` | `input_ids` int64 `(batch, sequence_length)`<br>`encoder_hidden_states` float32 `(batch, encoder_sequence_length, 512)`<br>`encoder_attention_mask` int64 `(batch, encoder_sequence_length)` | `logits` float32 `(batch, sequence_length, 32000)`          |

* `input_ids` 已包含 `[BOS(2)] ... [EOS(3)]`；`decoder` 起始符是 `decoder_start_token_id=2`。
* 生成时按**整段自回归**：每个 step 把已生成的完整 `input_ids` 再喂进去（无 KV Cache，**没有** `decoder_with_past_model.onnx`）。
* 跨 batch 的 padding 在右侧，`attention_mask` 置 0；mask 在**图内**构造，调用方不需要自己拼 mask。

### 2.7 本次真实校验结果（`hf_repo/export_meta.json`）

| 检查项                            | 结果                                                                                                                                                                                     |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onnx.checker`                 | 通过（encoder/decoder 均合法）                                                                                                                                                                |
| encoder `last_hidden_state` 逐值 | `max_abs_diff = 9.54e-07`，`allclose = true`                                                                                                                                            |
| decoder `logits` 逐值            | `max_abs_diff = 4.91e-05`，`allclose = true`                                                                                                                                            |
| decoder 每步 argmax 一致率          | `1.0`（32000 维 logits 的 argmax 完全一致）                                                                                                                                                    |
| 动态轴                            | 换到 `(batch=3, src=20)` / `(batch=3, tgt=12)` 后形状正确，`passed = true`                                                                                                                     |
| 权重文件                           | `model.safetensors`，参数量 **93,324,544（≈93.3 M）**，fp32                                                                                                                                   |
| HF 侧自检                         | `tokenizer_class = TransformerCustomTokenizer`、`model_class = TransformerForConditionalGeneration`、`tokenizer_ids_match = true`、`greedy_match = true`、`max_abs_logits_diff = 4.77e-05` |
| 翻译对照                           | HF 贪心 = ONNX 贪心 = “政府实施了诸多政策,改善国民生活水平。”；HF beam = ONNX beam = “政府实施了一系列政策改善国民的生活水平。”                                                                                                   |
| 导出后端 / 补丁                      | `dynamo(torch.export)`；`patch_applied = false`（原始模型直接可导，未改动任何语义）                                                                                                                       |

> 逐值误差在 `1e-5` 量级是 float32 下算子分解与累加顺序不同导致的正常现象；
> 对翻译真正关键的 **argmax 一致率为 1.0**，且端到端逐 token 完全一致。

---

## 3. 使用方接入

三种消费方式任选：**A. transformers（PyTorch + `generate()`）**、**B. 纯 onnxruntime（最小依赖）**、
**C. 直接用本工程的 `translate_onnx_hf.py`**。

> **环境约定：以下用法一律以「远程 HF 仓库」为环境**，仓库 id 固定为
> **`chou-lucas/transformer-en-zh-base`**（17 个文件，清单见 §4.2.1）。
> 本工程 `data/train/exp/weights/hf_repo` 只是**导出侧的本地产物**，消费端不需要它；
> 需要落地文件时统一用 `snapshot_download(REPO)`（默认落 HF 缓存，也可 `local_dir=` 指定），
> 拿到的目录与远端逐字节相同（§4.3.1 已逐文件校验 17/17）。

### 3.1 方式 A：transformers + `trust_remote_code=True`

```bash
pip install "transformers>=4.40" torch sentencepiece sacremoses safetensors
```

```python
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM

repo = "your-name/en-zh-base"          # 或本地目录 data/train/exp/weights/hf_repo
tokenizer = AutoTokenizer.from_pretrained(repo, trust_remote_code=True)   # trust_remote_code 不能省
model = AutoModelForSeq2SeqLM.from_pretrained(repo, trust_remote_code=True)
model.eval()

sentences = ["The government has implemented various policies to improve the living standards of its citizens."]
inputs = tokenizer(sentences, return_tensors="pt", padding=True)
out = model.generate(**inputs, max_length=60, num_beams=3)                 # use_cache 自动为 False
print(tokenizer.batch_decode(out, skip_special_tokens=True))
# ['政府实施了一系列政策改善国民的生活水平。']
```

* `tokenizer(...)` 会自动补 `BOS(2) + pieces + EOS(3)`（由自定义分词器的 `build_inputs_with_special_tokens` 负责）。
* `generation_config.json` 已经写好 `max_length=60` / `num_beams=3`，所以 `model.generate(**inputs)` 也能直接用默认值。
* 复制到本机加载时，把 `repo` 换成目录路径（如 `data/train/exp/weights/hf_repo`）即可，`auto_map` 依然生效。

### 3.2 方式 B：纯 onnxruntime（不需要 transformers 的模型代码）

只需要 `onnxruntime + numpy + sentencepiece`（+ `transformers` 仅为拿 tokenizer；也可以直接用 sentencepiece 手工编码）。

```python
import numpy as np
import onnxruntime as ort
from huggingface_hub import snapshot_download
from transformers import AutoTokenizer                       # 分词器仍用 HF 原生实现

REPO = "chou-lucas/transformer-en-zh-base"                   # 以远程仓库为环境
D = snapshot_download(REPO)                                  # 远程 -> 本地镜像（ONNX 需要真实文件路径）
PAD, BOS, EOS, MAXLEN = 0, 2, 3, 60

tok = AutoTokenizer.from_pretrained(REPO, trust_remote_code=True)      # 分词器同样来自远程仓库
enc = ort.InferenceSession(f"{D}/encoder_model.onnx", providers=["CPUExecutionProvider"])
dec = ort.InferenceSession(f"{D}/decoder_model.onnx", providers=["CPUExecutionProvider"])

input_ids = np.array([tok("I love you.")["input_ids"]], dtype=np.int64)   # 已含 BOS/EOS
attention_mask = (input_ids != PAD).astype(np.int64)         # 真实 token=1

memory = enc.run(None, {"input_ids": input_ids, "attention_mask": attention_mask})[0]          # (1,S,512)

cur = np.array([[BOS]], dtype=np.int64)                      # decoder_start_token_id
for _ in range(MAXLEN):
    logits = dec.run(None, {"input_ids": cur,
                            "encoder_hidden_states": memory,
                            "encoder_attention_mask": attention_mask})[0]
    nxt = int(logits[0, -1].argmax())
    cur = np.concatenate([cur, [[nxt]]], axis=1)
    if nxt == EOS:
        break

print(tok.decode(cur[0].tolist(), skip_special_tokens=True))  # 我爱你。（实测输出）
```

要点：batch / sequence_length 都是动态维，可以一次喂多条；只需做 **右侧 padding + attention_mask**，
mask 的细节由模型图自己处理。想换成 `log_probs` 输出（`--output log_probs`）时把 `argmax` 换成对 log 概率
累加即可（本工程 beam search 就是这么做的）。

### 3.3 方式 C：本工程的封装 [`translate_onnx_hf.py`](transformers_learning/translate_onnx_hf.py:815)

```bash
# 首选：直接以远程仓库为环境（内部 snapshot_download 后解码）
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --from-dev 3        # 交互式（q! 退出）
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --from-dev 5 --decode both
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --text "I love you." --show-details

# 下载验证（逐文件哈希 + Hub-HF / Hub-ONNX / 本地 PyTorch 三方逐 token 对照）
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --verify-hub --from-dev 3

# 离线/本地备选：换成 --onnx-dir <本地目录>（本工程导出目录，或 snapshot_download 的落地目录）
python translate_onnx_hf.py --onnx-dir data/train/exp/weights/onnx_hf
```

```python
# 当库用：仓库 id 与本地目录都支持
from translate_onnx_hf import Seq2SeqOnnxModel, one_sentence_translate

model = Seq2SeqOnnxModel.from_pretrained("chou-lucas/transformer-en-zh-base")   # 远程仓库（自动拉取）
# model = Seq2SeqOnnxModel.from_pretrained("data/train/exp/weights/onnx_hf")    # 或本地目录
print(model.generate("I love you.", num_beams=3))                        # ['我爱你。']
print(one_sentence_translate("I love you.", model, model.source_sp, model.target_sp,
                             model.bos_token_id, model.eos_token_id))    # 我爱你。
```

该封装提供了 [`Seq2SeqOnnxModel`](transformers_learning/translate_onnx_hf.py:71)：
`from_pretrained()`（自动读 `config.json` / `generation_config.json` / `tokenizer_config.json` 与 `.spm`）、
`encode()`、[`greedy_decode()`](transformers_learning/translate_onnx_hf.py:241)、
[`beam_search()`](transformers_learning/translate_onnx_hf.py:257)（beam 合成一个 batch 一次前向）、
[`generate()`](transformers_learning/translate_onnx_hf.py:305)；
以及函数签名与 [`translate_main.py`](transformers_learning/translate_main.py:14) 对齐的
[`translate()`](transformers_learning/translate_onnx_hf.py:323) / [`one_sentence_translate()`](transformers_learning/translate_onnx_hf.py:339)，
方便把老代码平滑切到 ONNX。

另外还支持**直接从 Hub 使用与校验**：`--repo-id <user/repo>` 会自动 `snapshot_download` 后走上面同一套解码；
加 `--verify-hub` 则执行「下载验证」（逐文件哈希 + Hub-HF / Hub-ONNX / 本地 PyTorch 三方逐 token 对照），
详见 §4.3。

### 3.4 上传与拉取

本工程的仓库已实际发布：**https://huggingface.co/chou-lucas/transformer-en-zh-base**（上传与下载验证的完整记录见 §4）。

```bash
# 上传：导出 + 上传 + 上传后校验（下载回本地重新校验）一步到位
export HF_TOKEN=hf_xxx                      # 或用 huggingface-cli login
python export_hf_repo.py --repo-id chou-lucas/transformer-en-zh-base \
                         --upload  chou-lucas/transformer-en-zh-base
# 跳过上传后校验：--no-upload-verify ｜ 私有仓库：--private ｜ 校验落地目录：--upload-verify-dir <dir>

# 也可以手动上传
hf upload chou-lucas/transformer-en-zh-base data/train/exp/weights/hf_repo

# 拉取
huggingface-cli download chou-lucas/transformer-en-zh-base --local-dir ./en-zh-base
```

```python
from huggingface_hub import snapshot_download
d = snapshot_download("chou-lucas/transformer-en-zh-base")                              # 默认落在 HF 缓存
d = snapshot_download("chou-lucas/transformer-en-zh-base", local_dir="./en-zh-base")    # 也可指定落地目录
# d 里的 17 个文件与远端逐字节相同（§4.3.1 已逐文件校验），之后按 §3.1 / §3.2 用这个路径加载
```

上传前建议检查：`README.md` 的 front matter、`config.json` 的 `auto_map` 路径与仓库内文件名一致、
`source.spm` / `target.spm` / `.py` 都在仓库根目录（脚本已保证），并自行补 `LICENSE`。

### 3.5 接入对齐清单

| 项                   | 要求                                                                 | 原因                                       |
| ------------------- | ------------------------------------------------------------------ | ---------------------------------------- |
| `trust_remote_code` | 加载模型与分词器**都要** `True`                                              | 自定义 `model_type` + 自定义分词器；否则加载失败或分词退化    |
| 特殊符号                | `pad=0, unk=1, bos=2, eos=3, decoder_start=2`                      | 与训练一致；写在各 `*_config.json` 里，可直接读         |
| 源句编码                | `[BOS] + sp.EncodeAsIds(text) + [EOS]`，**右侧** padding              | 训练时的 `collate_fn` 规则                     |
| `attention_mask`    | 真实 token = 1，padding = 0                                           | 图内据此构造 padding mask（不是拿 id 和 pad 比较）     |
| 词表                  | 编码用英文 spm、解码用中文 spm（`separate_vocabs=true`）                        | 两套独立 32k 词表                              |
| KV Cache            | 不要传 `use_cache=True`（无 cache 实现，`generation_config.json` 已置 false） | 图与模型都只支持整段自回归                            |
| 精度/EP               | 需要与 PyTorch 严格对齐时用 `CPUExecutionProvider`                          | CoreML/GPU 可能降精度（脚本默认 CPU，`--coreml` 可选） |
| 长度                  | `max_length=60`（训练时的 `config.max_len`）                             | 位置编码按该量级使用，过长无意义                         |
| 解码策略                | 质量优先用 `num_beams=3`，速度优先用贪心                                        | 训练/评测都是 beam=3                           |

### 3.6 常见问题（FAQ）

* **报 `model_type transformer_custom` 不认识 / 找不到配置类**：忘了 `trust_remote_code=True`，
  或仓库里缺少 `configuration_transformer_custom.py` / `modeling_transformer_custom.py`。
* **翻译结果明显变差、译文带英文**：分词器退化成了内置 `MarianTokenizer`（只补 EOS、用英文 spm 解码）。
  确认 `tokenizer_config.json` 的 `auto_map` 与 `tokenization_transformer_custom.py` 都在仓库里，并加 `trust_remote_code=True`。
* **`sentencepiece` / `.spm` 相关报错**：安装 `sentencepiece`，并确认 `source.spm` / `target.spm` 存在于目录中
  （`export_onnx_hf.py --no-tokenizer` 会跳过这两个文件、需要自己补；`export_hf_repo.py` 总是复制分词器资产）。
* **日志出现 “跳过 onnx.checker / 跳过数值校验”**：缺 `onnx` 或 `onnxruntime`，装上再重跑即可。
* **`dynamo` 导出失败并回退**：缺 `onnxscript`（`pip install onnxscript`），或显式用 `--exporter legacy`（只需 `onnx`）。
* **长句很慢**：没有 KV Cache，每生成一个 token 都要重算整段 decoder（约 0.1 s/step 量级，CPU）；
  想更快需要给模型加上 KV Cache 并导出 `decoder_with_past_model.onnx`（当前未实现）。
* **ONNX 文件和 safetensors 体积重复**：`hf_repo` 同时给 PyTorch 与 ONNX 两条路，若只服务其中一类，
  用 `--no-onnx` 或只跑 `export_onnx_hf.py`。
* **`--upload` 失败**：未登录或 token 无写权限；`huggingface-cli login` 后重试，或手动 `hf upload`。

---

## 4. 上传与下载验证（真实性复现记录）

> 本节是**本机真实执行过**的过程记录：命令、耗时、产物格式、校验方法与结论，可直接照抄复现。
> 相关代码：上传及校验落在 [`upload_repo()`](transformers_learning/export_hf_repo.py:540)（`--upload` 触发）；
> 下载验证落在 [`download_from_hub()`](transformers_learning/translate_onnx_hf.py:361) +
> [`verify_files_against_hub()`](transformers_learning/translate_onnx_hf.py:407) +
> [`verify_hub_download()`](transformers_learning/translate_onnx_hf.py:448)（`--verify-hub` 触发）。

### 4.1 发布结果

| 项                  | 值                                                                                                                          |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| 仓库                 | https://huggingface.co/chou-lucas/transformer-en-zh-base （public）                                                          |
| 首次上传 commit        | `549116932b89…`                                                                                                            |
| 回传校验报告后 commit     | `0c902ab05586…`                                                                                                            |
| 仓库文件数              | 17（本地导出的 16 个 + Hub 自动生成的 `.gitattributes`）                                                                                |
| 本地导出体积             | 16 个文件 / 756.0 MB（`model.safetensors` 375.56 + `decoder_model.onnx` 232.48 + `encoder_model.onnx` 145.29 + 其余小文件）          |
| safetensors（Hub 侧） | `{"total": 98444544, "parameters": {"F32": 98444544}}`（张量元素数；`model.parameters()` 统计为 93.3 M，差值来自 `amax/amin` 等非参数 buffer） |
| 上传耗时               | 首次约 155 s；本次幂等重传 5.2 s                                                                                                     |

### 4.2 上传过程与格式

```bash
cd transformers_learning
export HF_TOKEN=hf_xxx        # 或 huggingface-cli login；脚本不从仓库里读凭据
python export_hf_repo.py \
    --repo-id chou-lucas/transformer-en-zh-base \
    --upload  chou-lucas/transformer-en-zh-base \
    --upload-verify-dir /tmp/hf_upload_verify     # 上传后校验的落地目录（默认 HF 缓存）
    # 可选：--private ｜ --no-upload-verify ｜ --hub-token <token>
```

[`upload_repo()`](transformers_learning/export_hf_repo.py:540) 内部做了 6 件事：

1. 统计本地文件数与体积（16 个 / 756.0 MB）→ `create_repo(repo_type="model", exist_ok=True, private=…)`；
2. `upload_folder()` 整目录推送，**两类文件两种格式**（由 Hub 的 `.gitattributes` 规则决定）：
   * 普通 Git 提交（12 个）：`config.json`、`generation_config.json`、`tokenizer_config.json`、`special_tokens_map.json`、`vocab.json`、`target_vocab.json`、三个 `*.py`、`README.md`、`export_meta.json`——仓库里就是原文；
   * **LFS/Xet**（5 个）：`model.safetensors`、`encoder_model.onnx`、`decoder_model.onnx`、`source.spm`、`target.spm`——仓库里存的是指针，`resolve` / `snapshot_download` 会还原成真实权重（网页点下载拿到的也是完整文件）；
3. Hub 自动补 `.gitattributes`（这就是仓库比本地目录多 1 个文件的原因）；
4. 记录 `commit_revision`（`CommitInfo.oid`）——后续校验**钉在这个 revision 上**，保证验的就是刚推上去的那一版；
5. **上传后校验**：调用 [`verify_hub_download()`](transformers_learning/translate_onnx_hf.py:448)（流程见 §4.3），结果写进 `export_meta.json` 的 `upload` 字段；
6. 把带校验结论的 `export_meta.json` 再单独回传一次（几 KB），让**仓库自证**：`upload.commit_revision`、`upload.verified`、逐句对照明细都能在仓库里查到。

幂等性：本次重跑导出后上传，Hub 返回 `No files have been modified since last commit` 并跳过空提交——说明**导出是确定性的**（同样输入得到逐字节相同的产物）。

### 4.2.1 创建 repo 与上传后的仓库文件清单

`create_repo` 的请求返回 `POST /api/repos/create → HTTP/1.1 200 OK`（仓库已存在时 `exist_ok=True` 直接复用），
接着 `POST /api/models/<repo>/preupload/main → 200 OK`、`POST /api/models/<repo>/commit/main → 200 OK` 完成提交。

上传完成后，**远端仓库里就是下面这 17 个文件**（`HfApi().model_info(repo, files_metadata=True)` 实测，
revision `0c902ab05586`，合计 **756.05 MB**）：

| #   | 文件                                    | 大小        | 存储方式    | Hub 侧哈希（前 16 位）     | 作用                                                                               |
| --- | ------------------------------------- | --------- | ------- | ------------------- | -------------------------------------------------------------------------------- |
| 1   | `.gitattributes`                      | 1.6 KB    | Git     | `df42fd966679c0c1…` | Hub 自动生成：LFS 规则（`*.onnx`、`*.safetensors`，以及显式的 `source.spm` / `target.spm`）      |
| 2   | `README.md`                           | 6.5 KB    | Git     | `458672b4691b8edf…` | 模型卡（`library_name: transformers`、`pipeline_tag: translation`）+ 用法 + 校验摘要         |
| 3   | `config.json`                         | 820 B     | Git     | `d9d699adf796a401…` | 结构配置 + `auto_map`（`trust_remote_code` 的入口）                                       |
| 4   | `configuration_transformer_custom.py` | 3.4 KB    | Git     | `19cfb97d27ac5769…` | 自定义 Config 类 `TransformerCustomConfig`                                           |
| 5   | `decoder_model.onnx`                  | 232.48 MB | **LFS** | `d96a455f1d013d49…` | ONNX 解码器：`input_ids + encoder_hidden_states + encoder_attention_mask → logits`   |
| 6   | `encoder_model.onnx`                  | 145.29 MB | **LFS** | `b496fcf60f92a50b…` | ONNX 编码器：`input_ids + attention_mask → last_hidden_state`                        |
| 7   | `export_meta.json`                    | 13.4 KB   | Git     | `2a73eaaa63decfce…` | 导出 + 上传 + 校验报告（**仓库自证**的凭证）                                                      |
| 8   | `generation_config.json`              | 306 B     | Git     | `6f653163a59f3d6a…` | `max_length=60` / `num_beams=3` / `use_cache=false` / `decoder_start_token_id=2` |
| 9   | `model.safetensors`                   | 375.56 MB | **LFS** | `8c55df05c2d22d75…` | PyTorch 权重，93.3 M 参数，fp32                                                        |
| 10  | `modeling_transformer_custom.py`      | 18.9 KB   | Git     | `b36551fedf41df33…` | 自定义模型类（支持 `generate()`，无 KV Cache）                                               |
| 11  | `source.spm`                          | 0.76 MB   | **LFS** | `579e884a78d3230e…` | 英文 SentencePiece 模型                                                              |
| 12  | `special_tokens_map.json`             | 95 B      | Git     | `d3a99da4d9bc2a3a…` | `<s> </s> <unk> <pad>`                                                           |
| 13  | `target.spm`                          | 0.75 MB   | **LFS** | `64f82f91e04c5635…` | 中文 SentencePiece 模型                                                              |
| 14  | `target_vocab.json`                   | 0.57 MB   | Git     | `ddfba201cd9ce078…` | 中文词表（piece → id）                                                                 |
| 15  | `tokenization_transformer_custom.py`  | 1.6 KB    | Git     | `20871388e58b4470…` | 自定义分词器（`separate_vocabs` + 句首 BOS）                                               |
| 16  | `tokenizer_config.json`               | 621 B     | Git     | `81d7e267d13731e1…` | 分词器配置 + `auto_map`                                                               |
| 17  | `vocab.json`                          | 0.58 MB   | Git     | `209bcb6244831b8d…` | 英文词表（piece → id）                                                                 |

> 与本地导出目录的唯一差异：Hub 自动补的 `.gitattributes`（本地 16 个 → 仓库 17 个）。
> 走 LFS 的是 5 个文件（2 个 `*.onnx`、`model.safetensors`、`source.spm`、`target.spm`），
> 正是 `.gitattributes` 里声明的类型；其余 12 个是普通 Git 文件。

### 4.3 下载验证过程（消费端可独立复现）

```bash
# 方式一：本工程脚本一条命令完成「下载 + 校验 + 报告」
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --verify-hub --from-dev 3
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --verify-hub --no-hub-transformers   # 只做 ONNX 侧
# 方式二：直接用 Hub 上的模型翻译
python translate_onnx_hf.py --repo-id chou-lucas/transformer-en-zh-base --from-dev 3
```

[`verify_hub_download()`](transformers_learning/translate_onnx_hf.py:448) 的判定步骤与标准：

| #   | 步骤                                                                                        | 判定标准                                                                                                                                     |
| --- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `snapshot_download(repo_id, revision=<commit>)`                                           | 17 个文件全部落地到本地目录                                                                                                                          |
| 2   | **逐文件哈希**（[`verify_files_against_hub()`](transformers_learning/translate_onnx_hf.py:407)） | 大文件：本地 `sha256` == Hub `lfs.sha256`；小文件：本地 **git blob sha1** == Hub `blob_id`（不是只比体积）                                                    |
| 3   | ONNX 侧推理                                                                                  | `Seq2SeqOnnxModel.from_pretrained(下载目录)` 加载成功，贪心 + 3-beam 正常出结果                                                                          |
| 4   | transformers 侧推理                                                                          | `AutoTokenizer` / `AutoModelForSeq2SeqLM`.from_pretrained(repo_id, `trust_remote_code=True`) 加载成功；分词 id 与训练侧 `[BOS] + pieces + [EOS]` 一致 |
| 5   | 三路逐 token 对照                                                                              | Hub-HF、Hub-ONNX、本地 PyTorch 原模型（`--compare` 默认开）截断到 EOS 后 token 序列**完全相等**；HF 与 ONNX 的 logits `np.allclose`                               |
| 6   | 结论                                                                                        | 全通过 → `PASS ✅`、退出码 0；否则 `FAIL ❌`、退出码 1（报告见 [`print_hub_report()`](transformers_learning/translate_onnx_hf.py:560)）                       |

### 4.3.1 下载到本地后是否 OK：文件清单与逐文件校验

`snapshot_download("chou-lucas/transformer-en-zh-base", local_dir=…)` 在 revision `0c902ab05586` 拉下来的内容
（**17 个文件 + 1 个 HF 缓存目录，共 18 个条目**；LFS 指针已自动还原为真实权重，本次约 404 s / 790 MB）：

| #   | 下载后条目                                 | 大小            | 说明                                                                        |
| --- | ------------------------------------- | ------------- | ------------------------------------------------------------------------- |
| 1   | `.cache/`                             | 37 个文件        | huggingface_hub 的下载元数据（`.cache/huggingface/{trees,download}`），**不属于仓库内容** |
| 2   | `.gitattributes`                      | 1.6 KB        | 同远端                                                                       |
| 3   | `README.md`                           | 6.5 KB        | 同远端                                                                       |
| 4   | `config.json`                         | 820 B         | 同远端                                                                       |
| 5   | `configuration_transformer_custom.py` | 3.4 KB        | 同远端                                                                       |
| 6   | `decoder_model.onnx`                  | **232.48 MB** | LFS 已还原为真实权重                                                              |
| 7   | `encoder_model.onnx`                  | **145.29 MB** | LFS 已还原为真实权重                                                              |
| 8   | `export_meta.json`                    | 13.4 KB       | 含 `upload` 上传校验报告                                                         |
| 9   | `generation_config.json`              | 306 B         | 同远端                                                                       |
| 10  | `model.safetensors`                   | **375.56 MB** | LFS 已还原为真实权重                                                              |
| 11  | `modeling_transformer_custom.py`      | 18.9 KB       | 同远端                                                                       |
| 12  | `source.spm`                          | 0.76 MB       | 同远端                                                                       |
| 13  | `special_tokens_map.json`             | 95 B          | 同远端                                                                       |
| 14  | `target.spm`                          | 0.75 MB       | 同远端                                                                       |
| 15  | `target_vocab.json`                   | 0.57 MB       | 同远端                                                                       |
| 16  | `tokenization_transformer_custom.py`  | 1.6 KB        | 同远端                                                                       |
| 17  | `tokenizer_config.json`               | 621 B         | 同远端                                                                       |
| 18  | `vocab.json`                          | 0.58 MB       | 同远端                                                                       |

**结论：17/17 逐文件哈希一致、`all_match=True`**（抽样 6 个）：

| 文件                   | 大小        | 算法              | Hub 侧哈希             | 本地哈希                | 一致  |
| -------------------- | --------- | --------------- | ------------------- | ------------------- | --- |
| `model.safetensors`  | 375.56 MB | `lfs.sha256`    | `8c55df05c2d22d75…` | `8c55df05c2d22d75…` | ✅   |
| `decoder_model.onnx` | 232.48 MB | `lfs.sha256`    | `d96a455f1d013d49…` | `d96a455f1d013d49…` | ✅   |
| `encoder_model.onnx` | 145.29 MB | `lfs.sha256`    | `b496fcf60f92a50b…` | `b496fcf60f92a50b…` | ✅   |
| `source.spm`         | 0.76 MB   | `lfs.sha256`    | `579e884a78d3230e…` | `579e884a78d3230e…` | ✅   |
| `config.json`        | 820 B     | `git-blob-sha1` | `d9d699adf796a401…` | `d9d699adf796a401…` | ✅   |
| `README.md`          | 6.5 KB    | `git-blob-sha1` | `458672b4691b8edf…` | `458672b4691b8edf…` | ✅   |

这份下载下来的目录还通过了三方逐 token 对照（§4.3 表格第 3–5 步）：
`transformers(trust_remote_code) + generate`、`onnxruntime`、本地 PyTorch 原模型三路输出完全一致，
并且 `AutoTokenizer` 给出的 id 与训练侧 `[BOS] + pieces + [EOS]` 完全相同。

> **这个校验不是走过场**：拿一份**旧快照**（在 revision `54911693…` 下载，而 main 之后推进到了 `0c902ab0…`）去比，
> 会准确报出 **16/17**——唯一不一致的正是回传更新过的 `export_meta.json`；重新 `snapshot_download` 到当前 revision 后恢复 17/17。
> 说明它做的是**逐字节**比对（`lfs.sha256` / git blob sha1），不是只比体积或文件名。

### 4.4 本次真实校验数据

| 阶段           | 检查项                                                     | 结果                                                                                                         |
| ------------ | ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 导出（上传前，§2.7） | `onnx.checker` ｜ encoder 逐值 ｜ decoder 逐值 ｜ argmax ｜ 动态轴 | 通过 ｜ 9.54e-07 ｜ 4.91e-05 ｜ 1.0 ｜ passed                                                                    |
| 导出（上传前）      | HF `trust_remote_code` 自检（2 句）                          | 分词一致 ✅、贪心逐 token 一致 ✅、beam 一致 ✅、logits diff 4.77e-05 / 6.68e-05                                            |
| 上传           | 16 文件 / 756.0 MB → commit `549116932b89`                | 成功；重跑内容相同 → Hub 跳过空提交                                                                                      |
| 上传后校验（下载回本地） | 逐文件哈希                                                   | **17/17 一致**（5 个 LFS 文件比 `lfs.sha256`：2 个 `*.onnx`、`model.safetensors`、2 个 `*.spm`；其余 12 个比 git blob sha1） |
| 上传后校验        | Hub-HF vs Hub-ONNX vs 本地 PyTorch                        | **3/3 句逐 token 完全一致**；logits max_abs_diff 5.15e-05 / 6.68e-05 / 4.20e-05                                   |
| 上传后校验        | 模型规模                                                    | `TransformerCustomTokenizer` / `TransformerForConditionalGeneration`，93.3 M 参数                             |
| 结论           | —                                                       | **PASS ✅**                                                                                                 |

逐句对照（贪心；`beam=3` 时 Hub-HF 与 Hub-ONNX 同样逐字相同）：

| #   | 英文（截断）                                             | Hub HF `generate()`                        | Hub ONNX | 本地 PyTorch |
| --- | -------------------------------------------------- | ------------------------------------------ | -------- | ---------- |
| 1   | Some analysts argue that the negative effects…     | 一些分析家认为这种结果的负面影响仅仅局限于“几个月”。                | 同左       | 同左         |
| 2   | The Fed apparently could not stomach the sell-off… | 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。 | 同左       | 同左         |
| 3   | Renewing the South Korean Miracle                  | 开发韩国奇迹                                     | 同左       | 同左         |

### 4.5 注意事项

* **不要把 token 写进仓库或文档**：脚本只从 `--hub-token` / 环境变量 `HF_TOKEN` 读取，上传内容不含任何凭据（本次会话中明文出现过的 token，建议到 https://huggingface.co/settings/tokens 轮换）。
* `export_meta.json` 里的 `checkpoint` 记录的是相对路径（`data/train/exp/weights/best_bleu_26.30.pth`），不会泄露本机绝对路径。
* 上传后校验会再下载约 790 MB；网络受限时用 `--no-upload-verify`，或 `--no-hub-transformers` 只做 ONNX 侧校验。
* 上传失败或校验不通过时 `export_hf_repo.py` 退出码为 1（本地导出产物仍然完整）；需要私有仓库加 `--private`。
* 建议为仓库补一个 `LICENSE`（导出脚本不会自动生成）。

---

### 4.6 示例必须写真实输出（一个反面案例）

"实现一致性"与"翻译质量"是两件事。以早期示例里的句子 `The cat is sleeping on the sofa.` 为例，
同样输入在四条路径上的**实测**输出：

| 实现路径                                                             | 输出                                             |
| ---------------------------------------------------------------- | ---------------------------------------------- |
| 本工程 ONNX 贪心（`Seq2SeqOnnxModel.greedy_decode`）                    | `狗在 ⁇ 里死。`                                     |
| 本地 PyTorch 原模型贪心（同一份 `.pth`）                                     | `狗在 ⁇ 里死。`                                     |
| 远程仓库 ONNX 3-beam                                                 | ` ⁇ 子在沙子中睡觉。`                                  |
| 远程仓库 transformers（`trust_remote_code` + `generate(num_beams=3)`） | `子在沙子中睡觉。`（`skip_special_tokens=True` 会吃掉 `⁇`） |

结论两条：

1. **等价性没问题**：ONNX 与 HF/PyTorch 逐 token 一致，同路径下贪心/beam 也彼此一致 —— 图与权重是等价的；
2. **质量确实差**：`cat` 译成"狗"、并出现 `⁇`（U+2047，SentencePiece 的未知片段占位）。
   这属于**模型与领域**问题：本模型在这份偏新闻/书面语的 25k 平行语料上训到 BLEU ≈ 26，
   短口语句（`The cat …` 这类）本就不在分布内，与 ONNX 导出无关。

因此本文档与脚本中的示例输出**一律使用实测真值**，不再沿用旧注释里未经核实的"期望译文"，
例如：`I love you.` → `我爱你。`；`The government has implemented various policies to improve the
living standards of its citizens.` → `政府实施了诸多政策,改善国民生活水平。`

---

## 5. 原始证据（日志 / 输出 / 代码片段）

> 本节把上面所有结论的**原始输出**贴出来，便于读者核对，而不是"我说结论你信我"。
> 日志均为本机实际执行的终端输出（时间戳为本地时间 UTC+8），代码片段摘自当前仓库文件。

### 5.1 导出 + 首次上传的日志（节选）

```text
2026-09-23 11:56:16,605-export_onnx_hf-INFO-导出成功 [dynamo(torch.export)] -> .../hf_repo/decoder_model.onnx (232.5 MB)
2026-09-23 11:56:17,866-export_onnx_hf-INFO-[校验] onnx.checker 通过（encoder_model.onnx, decoder_model.onnx）
2026-09-23 11:56:18,080-export_onnx_hf-INFO-[校验] last_hidden_state  shape=[2, 16, 512] max_abs_diff=9.537e-07 max_rel_diff=5.412e-02 通过=True
2026-09-23 11:56:18,084-export_onnx_hf-INFO-[校验] logits             shape=[2, 16, 32000] max_abs_diff=4.911e-05 max_rel_diff=1.450e+01 通过=True
2026-09-23 11:56:18,085-export_onnx_hf-INFO-[校验] decoder 每步 argmax 一致率: 1.000000
2026-09-23 11:56:18,095-export_onnx_hf-INFO-[校验] 动态轴: encoder (3, 20) -> [3, 20, 512], decoder (3, 12) -> [3, 12, 32000]  通过=True
2026-09-23 11:56:19,272-export_hf_repo-INFO-已写出 vocab.json（32000 个 piece）
2026-09-23 11:56:21,067-export_hf_repo-INFO-[自检] TransformerCustomTokenizer / TransformerForConditionalGeneration，参数量 93.3M
2026-09-23 11:56:21,975-export_hf_repo-INFO-[自检]   分词一致=True | 贪心逐 token 一致=True | logits max_abs_diff=5.150e-05
2026-09-23 11:56:21,975-export_hf_repo-INFO-[自检]   HF : 一些分析家认为这种结果的负面影响仅仅局限于“几个月”。
2026-09-23 11:56:21,975-export_hf_repo-INFO-[自检]   ONNX: 一些分析家认为这种结果的负面影响仅仅局限于“几个月”。
2026-09-23 11:56:21,975-export_hf_repo-INFO-[自检]   HF beam  : 一些分析家认为,这一结果的消极效应对“几个月”只能维持。
2026-09-23 11:56:21,975-export_hf_repo-INFO-[自检]   ONNX beam: 一些分析家认为,这一结果的消极效应对“几个月”只能维持。
2026-09-23 11:56:24,110-httpx-INFO-HTTP Request: POST https://huggingface.co/api/repos/create "HTTP/1.1 200 OK"
2026-09-23 11:56:24,517-httpx-INFO-HTTP Request: POST https://huggingface.co/api/validate-yaml "HTTP/1.1 200 OK"
2026-09-23 11:56:25,029-httpx-INFO-HTTP Request: POST https://huggingface.co/api/models/chou-lucas/transformer-en-zh-base/preupload/main "HTTP/1.1 200 OK"
2026-09-23 11:58:59,656-httpx-INFO-HTTP Request: POST https://huggingface.co/api/models/chou-lucas/transformer-en-zh-base/commit/main "HTTP/1.1 200 OK"
2026-09-23 11:58:59,657-export_hf_repo-INFO-已上传: https://huggingface.co/chou-lucas/transformer-en-zh-base
```

### 5.2 重跑导出 + 上传 + 上传后校验的日志（幂等重传）

```text
2026-09-23 12:20:26,593-export_hf_repo-INFO-开始上传 chou-lucas/transformer-en-zh-base（16 个文件，756.0 MB）
2026-09-23 12:20:32,211-huggingface_hub._upload_pipeline-WARNING-No files have been modified since last commit. Skipping to prevent empty commit.
2026-09-23 12:20:32,563-export_hf_repo-INFO-上传完成（5.2s）: https://huggingface.co/chou-lucas/transformer-en-zh-base
2026-09-23 12:20:32,563-export_hf_repo-INFO-[上传校验] 从 Hub 下载回来重新校验：逐文件哈希 + HF/ONNX/本地 PyTorch 逐 token 对照 ...
2026-09-23 12:21:34,960-translate_onnx_hf-INFO-已从 Hub 拉取 chou-lucas/transformer-en-zh-base@549116932b89 -> /private/tmp/hf_upload_verify（17 个文件）
2026-09-23 12:21:35,808-translate_onnx_hf-INFO-[下载验证] 文件哈希: 17/17 一致（revision=549116932b89）
2026-09-23 12:21:35,931-translate_onnx_hf-INFO-onnxruntime 1.30.0 已加载 /private/tmp/hf_upload_verify（output_mode=logits, providers=['CPUExecutionProvider']）
2026-09-23 12:21:35,931-translate_onnx_hf-INFO-模型输入: encoder ['input_ids', 'attention_mask'] / decoder ['input_ids', 'encoder_hidden_states', 'encoder_attention_mask']
2026-09-23 12:21:37,151-translate_onnx_hf-INFO-[下载验证] transformers 侧: TransformerCustomTokenizer / TransformerForConditionalGeneration（93.3M 参数）
2026-09-23 12:21:39,444-translate_onnx_hf-INFO-[下载验证] The Fed apparently could not stomach the sell-off in global
2026-09-23 12:21:39,444-translate_onnx_hf-INFO-[下载验证]   Hub HF  : 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。
2026-09-23 12:21:39,444-translate_onnx_hf-INFO-[下载验证]   Hub ONNX: 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。
2026-09-23 12:21:39,444-translate_onnx_hf-INFO-[下载验证]   本地原始: 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。
2026-09-23 12:21:39,457-export_hf_repo-INFO-[上传校验] PASS ✅ https://huggingface.co/chou-lucas/transformer-en-zh-base 可加载、可推理，且与本地 PyTorch 逐 token 一致
2026-09-23 12:21:41,730-export_hf_repo-INFO-已把上传校验报告同步进仓库的 export_meta.json: 0c902ab05586
```

### 5.3 下载验证命令的原始输出（`--repo-id … --verify-hub`）

```text
仓库        : chou-lucas/transformer-en-zh-base  (private=False, revision=549116932b89)
拉取目录    : /private/tmp/hf_dl_verify   文件数: 17
safetensors : {'total': 98444544, 'parameters': {'F32': 98444544}}
文件哈希    : 17/17 与 Hub 元数据一致（大文件 lfs.sha256，小文件 git blob sha1）
transformers: TransformerCustomTokenizer / TransformerForConditionalGeneration (93.3M 参数)
----------------------------------------------------------------------------------------------------
[1] EN            : Some analysts argue that the negative effects of such an out
    Hub HF        : 一些分析家认为这种结果的负面影响仅仅局限于“几个月”。
    Hub ONNX      : 一些分析家认为这种结果的负面影响仅仅局限于“几个月”。
    本地 PyTorch  : 一些分析家认为这种结果的负面影响仅仅局限于“几个月”。
    判定          : HF==ONNX True | ONNX==本地PyTorch True | logits max_abs_diff 5.150e-05
    束搜索(3)     : 一些分析家认为,这一结果的消极效应对“几个月”只能维持。
----------------------------------------------------------------------------------------------------
[2] EN            : The Fed apparently could not stomach the sell-off in global financial markets in Jan
    Hub HF        : 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。
    Hub ONNX      : 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。
    本地 PyTorch  : 美联储显然不能忍受1月到2日的全球金融市场抛售量,而这一下跌主要受进一步收紧的担忧。
    判定          : HF==ONNX True | ONNX==本地PyTorch True | logits max_abs_diff 6.676e-05
----------------------------------------------------------------------------------------------------
[3] EN            : Renewing the South Korean Miracle
    Hub HF        : 开发韩国奇迹
    Hub ONNX      : 开发韩国奇迹
    本地 PyTorch  : 开发韩国奇迹
    判定          : HF==ONNX True | ONNX==本地PyTorch True | logits max_abs_diff 4.196e-05
====================================================================================================
下载验证结论: PASS ✅ 从 Hub 拉取的仓库可加载、可推理，且与本地 PyTorch 逐 token 一致
```

### 5.4 逐文件哈希比对的原始输出（含"旧快照会被抓出来"的对照）

```text
# ① 旧快照：下载于 revision 54911693…，而 main 已推进到 0c902ab0…（export_meta.json 已被更新）
revision: 0c902ab05586 | all_match: False | files: 17
| `README.md`          | 0.01 MB   | git-blob-sha1 | `458672b4691b8edf…` | `458672b4691b8edf…` | True |
| `config.json`        | 0.00 MB   | git-blob-sha1 | `d9d699adf796a401…` | `d9d699adf796a401…` | True |
| `decoder_model.onnx` | 232.48 MB | lfs.sha256    | `d96a455f1d013d49…` | `d96a455f1d013d49…` | True |
| `encoder_model.onnx` | 145.29 MB | lfs.sha256    | `b496fcf60f92a50b…` | `b496fcf60f92a50b…` | True |
| `model.safetensors`  | 375.56 MB | lfs.sha256    | `8c55df05c2d22d75…` | `8c55df05c2d22d75…` | True |
| `source.spm`         | 0.76 MB   | lfs.sha256    | `579e884a78d3230e…` | `579e884a78d3230e…` | True |

# ② 重新 snapshot_download 到当前 revision 之后
下载耗时 404.0s | revision 0c902ab05586 | 文件数 17
逐文件哈希: 17/17 一致 | all_match=True | 不一致=无
下载后目录条目数: 18（含 .cache 目录）
条目: .cache, .gitattributes, README.md, config.json, configuration_transformer_custom.py,
      decoder_model.onnx, encoder_model.onnx, export_meta.json, generation_config.json,
      model.safetensors, modeling_transformer_custom.py, source.spm, special_tokens_map.json,
      target.spm, target_vocab.json, tokenization_transformer_custom.py, tokenizer_config.json, vocab.json
```

① 与 ② 的差别只有 1 个文件（`export_meta.json`），恰好印证了"逐字节比对"是真的在比对。

### 5.5 关键代码片段

```python
# export_hf_repo.py:540  upload_repo()：创建 repo -> 上传 -> 记录 revision -> 下载回来校验 -> 结论入库
create_repo(repo_id, repo_type="model", exist_ok=True, private=bool(args.private), token=token)
commit = HfApi(token=token).upload_folder(
    repo_id=repo_id, folder_path=repo_dir, token=token,
    commit_message="Upload transformer en-zh (safetensors + ONNX + custom code)")
report["commit_revision"] = getattr(commit, "oid", None) or HfApi(token=token).model_info(repo_id).sha
...
verification = verify_hub_download(
    repo_id, sentences, max_len=int(config.max_len), beam_size=int(config.beam_size),
    revision=report["commit_revision"],            # 钉在刚上传的那个 commit 上校验
    download_dir=args.upload_verify_dir, token=token,
    ckpt=resolve_path(args.ckpt), with_transformers=True, coreml=args.coreml)
report["verified"] = bool(verification["all_matched"])
```

```python
# translate_onnx_hf.py:407  verify_files_against_hub()：逐文件哈希（LFS 比 sha256，普通文件比 git blob sha1）
if sibling.lfs is not None:
    algo, local_hash, remote_hash = "lfs.sha256", _sha256_file(path), sibling.lfs.sha256
else:
    algo, local_hash, remote_hash = "git-blob-sha1", _git_blob_sha1(path), sibling.blob_id
match = bool(remote_hash is None or local_hash == remote_hash)
```

```python
# translate_onnx_hf.py:605  load_reference_model()：把 .pth 装回本工程模型，作为第三路对照
state_dict = torch.load(ckpt_path, map_location="cpu", weights_only=True)
if isinstance(state_dict, dict) and isinstance(state_dict.get("state_dict"), dict):
    state_dict = state_dict["state_dict"]
model.load_state_dict({k[7:] if k.startswith("module.") else k: v
                       for k, v in state_dict.items()})
model.eval()
```

### 5.6 文档自身的机械自检（读者可一键复现）

```text
$ ~/.penv/bin/python - <<'PY'     # 校验文档里所有链接：文件存在 + 行号命中 def/class + 代码围栏闭合
[文档自检] 行数=820 围栏=44(闭合=True) 引用=43 问题=无 ✅
PY
$ ~/.penv/bin/python -m py_compile export_hf_repo.py export_onnx_hf.py translate_onnx_hf.py translate_onnx.py export_onnx.py
OK ✅
```

---

## 附录：关键代码索引

| 位置                                                                                                                               | 说明                                                                |
| -------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| [`export_onnx_hf.py:134`](transformers_learning/export_onnx_hf.py:134) / [`139`](transformers_learning/export_onnx_hf.py:139)    | 图内 mask 构造（与 `tools/data_loader.py` 一致）                           |
| [`export_onnx_hf.py:150`](transformers_learning/export_onnx_hf.py:150)                                                           | ONNX 友好化补丁（仅作兜底，本次未启用）                                            |
| [`export_onnx_hf.py:175`](transformers_learning/export_onnx_hf.py:175) / [`187`](transformers_learning/export_onnx_hf.py:187)    | `HFEncoderModel` / `HFDecoderModel` 包装                            |
| [`export_onnx_hf.py:343`](transformers_learning/export_onnx_hf.py:343)                                                           | 导出后端选择：dynamo → legacy 回退                                         |
| [`export_onnx_hf.py:448`](transformers_learning/export_onnx_hf.py:448)                                                           | `verify_onnx`：checker + 逐值 + argmax + 动态轴                         |
| [`export_onnx_hf.py:729`](transformers_learning/export_onnx_hf.py:729)                                                           | `OnnxTranslator`（`from_pretrained` / `generate`）                  |
| [`export_hf_repo.py:122`](transformers_learning/export_hf_repo.py:122)                                                           | `config.json` 构造（含 `auto_map`）                                    |
| [`export_hf_repo.py:217`](transformers_learning/export_hf_repo.py:217)                                                           | 分词器资产：`.spm` 复制 + `vocab.json` 导出                                 |
| [`export_hf_repo.py:236`](transformers_learning/export_hf_repo.py:236)                                                           | 模型卡生成（front matter + 用法 + 校验摘要）                                   |
| [`export_hf_repo.py:390`](transformers_learning/export_hf_repo.py:390)                                                           | `verify_hf_repo`：模拟别人 `trust_remote_code` 加载 + `generate`         |
| [`export_hf_repo.py:594`](transformers_learning/export_hf_repo.py:594)                                                           | 主流程（含上传、上传后校验、meta 回传）                                            |
| [`export_hf_repo.py:540`](transformers_learning/export_hf_repo.py:540)                                                           | `upload_repo()`：上传 + 上传后校验（下载回本地重新校验）                             |
| [`translate_onnx_hf.py:71`](transformers_learning/translate_onnx_hf.py:71)                                                       | `Seq2SeqOnnxModel`（纯 onnxruntime 封装）                              |
| [`translate_onnx_hf.py:361`](transformers_learning/translate_onnx_hf.py:361)                                                     | `download_from_hub()`：`snapshot_download` + 仓库元信息                 |
| [`translate_onnx_hf.py:407`](transformers_learning/translate_onnx_hf.py:407)                                                     | `verify_files_against_hub()`：逐文件哈希（lfs.sha256 / git blob sha1）    |
| [`translate_onnx_hf.py:448`](transformers_learning/translate_onnx_hf.py:448)                                                     | `verify_hub_download()`：下载验证主流程                                   |
| [`translate_onnx_hf.py:560`](transformers_learning/translate_onnx_hf.py:560)                                                     | `print_hub_report()`：下载验证报告打印                                     |
| [`hf_template/modeling_transformer_custom.py:288`](transformers_learning/hf_template/modeling_transformer_custom.py:288)         | `TransformerForConditionalGeneration`（支持 `generate()`，无 KV Cache） |
| [`hf_template/tokenization_transformer_custom.py:25`](transformers_learning/hf_template/tokenization_transformer_custom.py:25)   | `TransformerCustomTokenizer`（separate vocabs + 句首 BOS）            |
| [`hf_template/configuration_transformer_custom.py:26`](transformers_learning/hf_template/configuration_transformer_custom.py:26) | `TransformerCustomConfig`                                         |