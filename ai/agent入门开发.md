# Agent 入门开发实录（DeepSeek：控制台 Agent + 手机网页聊天）

> 本文记录 [`agent/`](agent/README.md:1) 这个入门 demo 的**完整开发过程**：
> 
> 需求拆解 → 方案设计 →
> 逐文件实现 → 真实联调 → 踩坑与修复 → 二次开发指南。
> 对照代码阅读效果最好，所有关键位置都带 `文件:行号` 链接。
> 
> 项目有**两个入口，同一目录、同一个 `llm.py`、同一份 `agent/.env`**：
> 
> 🖥️ `python3 agent/main.py` —— 控制台 Agent（「命令: / 完成:」协议 + 本机执行命令）；
> 
> 📱 `python3 agent/web.py` —— 手机网页聊天（SSE 流式 + 可选语音朗读）。
> 
> 本文基于 deepseek api进行开发，当然也可以自己部署个推理服务。

---

## 0. 一页速览

| 项   | 内容                                                                                                                   |
| --- | -------------------------------------------------------------------------------------------------------------------- |
| 目标  | 写一个能让初学者看懂「Agent 是怎么转起来的」的最小可用 demo                                                                                  |
| 形态  | ① 终端 REPL（`while` 读输入）+ 模型决策循环 + 本机执行 shell 命令；② 手机网页聊天（SSE 流式 + 语音朗读）                                               |
| LLM | DeepSeek（OpenAI 兼容`/chat/completions`，SSE 流式），两个入口**共用** [`llm.py`](agent/llm.py:1)                                  |
| 依赖  | **零第三方依赖**（只用标准库 `urllib` / `os.popen` / `http.server`）                                                              |
| 协议  | 控制台版：模型只能输出`命令:XXX` 或 `完成:XXX`；网页版：纯聊天，不执行命令                                                                         |
| 配置  | [`agent.md`](agent/agent.md:1)（人设 + 协议）+ [`skill.md`](agent/skill.md:1)（技能文档：百度查询）+ [`config.py`](agent/config.py:1) |
| 代码量 | 6 个 Python 文件（934 行）+ 3 个前端文件（403 行）                                                                                 |

跑起来只要三条命令：

```bash
cd /Users/lucas/PycharmProjects/ai
cp agent/.env.example agent/.env      # 填入 DEEPSEEK_API_KEY
python3 agent/main.py                 # 🖥️ 控制台版
python3 agent/web.py                  # 📱 网页版（启动日志里会打印手机访问地址）
```

---

## 1. 需求与验收对照

| #    | 原始需求                         | 实现方式                                                                | 位置                                                                          |
| ---- | ---------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| 1    | 目录为`agent`                   | 独立目录，可单独拷走运行                                                        | [`agent/`](agent/)                                                          |
| 2    | 支持`agent.md`、`skill.md` 两个示例 | `agent.md`=Agent 定义（front-matter + 人设 + 协议）；`skill.md`=技能文档（百度网页查询） | [`agent.md`](agent/agent.md:1)、[`skill.md`](agent/skill.md:1)               |
| 3    | 基于`while` 用户 input 交互        | 外层`while True` + `input()`；内层命令循环；支持 `/help /skills /clear /exit`   | [`main()`](agent/main.py:192)、[`run_turn()`](agent/main.py:104)             |
| 4    | 用 DeepSeek API 作为 LLM        | 纯标准库客户端，流式 SSE，支持`deepseek-chat` / `deepseek-reasoner`              | [`DeepSeekLLM`](agent/llm.py:33)、[`chat()`](agent/llm.py:126)               |
| 追加 1 | 提示词要「命令:/完成:」二选一格式           | 原文写进`agent.md`，并在 system prompt 末尾二次强调                              | [`agent.md`](agent/agent.md:14)、[`build_system_prompt()`](agent/main.py:77) |
| 追加 2 | skill/tool 命令用`os.popen` 执行  | 命令执行层唯一入口                                                           | [`run_command()`](agent/tools.py:46)                                        |
| 追加 3 | `skill.md` 改为「网上百度查询」        | curl 抓百度 + HTMLParser 解析 + 三级反爬兜底                                   | [`skill.md`](agent/skill.md:35)                                             |
| 追加 4 | 做一个网页版聊天，手机输入 URL 就能用        | 标准库 HTTP 服务 + SSE 流式 + 移动端适配界面                                      | [`web.py`](agent/web.py:1)、[`static/`](agent/static/index.html:1)           |
| 追加 5 | 可选「把文字用语音读出来」                | 浏览器原生`speechSynthesis`，顶栏「朗读回复」开关，免 Key                             | [`app.js`](agent/static/app.js:1)                                           |
| 追加 6 | 控制台版与网页版代码放在一起、LLM 请求用同一个    | 两个入口都在`agent/`，共用 `llm.py` / `config.py` / `.env`                   | [`llm.py`](agent/llm.py:1)、[`config.py`](agent/config.py:1)                 |
| 追加 7 | 网页只保留聊天功能                    | 只做「发消息 + 流式回复 + 朗读」，去掉设置面板、历史、语音输入等                                 | [`index.html`](agent/static/index.html:1)                                   |

---

## 2. 目录结构

```text
agent/
├── agent.md      # Agent 定义：front-matter(name/model/temperature) + 人设 + 输出协议 + 约束
├── skill.md      # Skill 示例：baidu_search（何时用/前置/步骤/兜底/输出要求）
├── main.py       # 🖥️ 控制台版入口：外层 while 交互 + 内层命令循环 + 协议解析 + 预算强制收尾
├── web.py        # 📱 网页版入口：标准库 HTTP 服务 + SSE 流式聊天
├── static/       # 网页版前端：index.html / style.css / app.js
├── llm.py        # ⭐ DeepSeek 客户端（唯一实现）：chat() / stream() / complete()
├── config.py     # ⭐ 共用配置：路径 + .env + 预算 + 网页端口与人设
├── tools.py      # 控制台版专用：os.popen 执行 + 危险命令黑名单 + 输出截断
├── skills.py     # 控制台版专用：技能发现（markdown front-matter → 技能清单）
├── .env.example  # 复制为 .env 并填入 DEEPSEEK_API_KEY（两个入口共用）
└── README.md     # 使用说明（面向使用者）
```

代码已经分享在：[https://github.com/Zeb-D/ai/tree/main/agent](https://github.com/Zeb-D/ai/tree/main/agent)  
请一键三连，谢谢～，如有困惑可一起v我交流

### 2.1 控制台演示

#### 正常逻辑

```text
/Users/lucas/.penv/bin/python /Users/lucas/PycharmProjects/ai/agent/main.py 

╭──────────────────────────────────────────────╮
│   🐣 入门 Agent Demo（DeepSeek + 命令协议）    │
╰──────────────────────────────────────────────╯

模型：deepseek-chat    工作目录：/Users/lucas/PycharmProjects/ai
技能：baidu_search    输入 /help 查看命令，/exit 退出

👤 你 > 1+1 = ？

🤖 入门助手 > 完成:1+1 = 2

👤 你 > 再加10

🤖 入门助手 > 完成:2 + 10 = 12
```

#### 脚本命令

```text
👤 你 > 查看当前目录列表

🤖 入门助手 > 命令:ls -la
   [1/10] $ ls -la
   ↳ total 2216 drwxr-xr-x@ 20 lucas staff 640 Sep 24 01:51 . drwxr-xr-x 31 lucas staff 992 Sep 21 19:30 .. -rw-r--r--@ 1 lucas staff 10244 Sep 23 12:32 .DS_Store drwxr-xr-x@ 15 lucas staff 480 Sep 11 16:44 .git -rw-r--r--@ 1 lucas staff 190 Jun 13 2025 .gitignore drwxr-xr-x@ 10 lucas staff 320 Sep 24 00...

🤖 入门助手 > 完成:当前目录 `/Users/lucas/PycharmProjects/ai` 列表如下：

**目录**
- `agent`、`common`、`deepLearning`、`llama`、`llama-2-70b`、`ml`、`sort`、`transformers_learning`
- 隐藏目录：`.git`、`.idea`、`.pytest_cache`

**文件**
- `agent入门开发.md`（32 KB）、`README.md`、`main.py`、`.gitignore`、`.python-version`
- `transformer算法原理-S.pdf`（约 1.0 MB）

共 20 个条目（含 `.` 和 `..`）。
```

#### 网上查询

```text
👤 你 > deepseek 最新模型版本，给出 3 条来源链接

🤖 入门助手 > 命令:cat agent/skill.md
   [1/10] $ cat agent/skill.md
   ↳ --- type: skill name: baidu_search description: "用 curl 访问百度网页搜索，抓取并解析搜索结果的标题/链接/摘要（纯命令行实现）" tags: [搜索, 百度, 网络, curl] --- # 技能：百度网页搜索（baidu_search） > 本文件是「技能文档」示例：Agent 通过 `cat agent/skill.md` 读到它，然后照着下面的命令一步步执行。 > 技能文档不写代码逻辑，只写「用哪些命令、按什么顺序、看什么输出、失败了怎么办」。 ## 何时使用 - 用户说「百度一下 XXX」「网上查一下 XXX」「搜一下最新的 XX...

🤖 入门助手 > 命令:date '+%Y-%m-%d %H:%M'; python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' 'deepseek 最新模型版本'
   [2/10] $ date '+%Y-%m-%d %H:%M'; python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' 'deepseek 最新模型版本'
   ↳ 2026-09-24 02:06 deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC

🤖 入门助手 > 命令:cat > /tmp/baidu_parse.py <<'PY'
import re, sys
from html import unescape
from html.parser import HTMLParser
from urllib.parse import unquote

path = sys.argv[1] if len(sys.argv) > 1 else "/tmp/baidu_search.html"
raw = open(path, encoding="utf-8", errors="ignore").read()


class _Text(HTMLParser):
    def __init__(self):
        super().__init__(convert_charrefs=True)
        self.parts = []

    def handle_data(self, data):
        self.parts.append(data)


def plain(text):
    text = re.sub(r"(?is)<(script|style).*?</\1>", " ", text)
    text = re.sub(r"<!--.*?-->", " ", text, flags=re.S)
    parser = _Text()
    parser.feed(text)
    return re.sub(r"\s+", " ", " ".join(parser.parts)).strip()


def real_url(block):
    match = re.search(r"(https?%3A%2F%2F[^\"&\\ ]+)", unescape(block))
    return unquote(match.group(1)) if match else ""


print(f"# 页面 {len(raw)} 字符")
for i, match in enumerate(re.finditer(r"<h3[^>]*>(.*?)</h3>", raw, re.S), 1):
    if i > 10:
        break
    block = match.group(1)
    title = plain(block)
    if not title:
        continue
    jump = re.search(r'href="(https?://[^"]+)"', block)
    print(f"{i}. {title}")
    if jump:
        print(f"   链接: {jump.group(1)}")
    if real := real_url(block):
        print(f"   真实地址: {real}")
    print(f"   摘要: {plain(raw[match.end(): match.end() + 2500])[:140]}")
PY
UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'; curl -sL --compressed -A "$UA" -H 'Referer: https://www.baidu.com/' -H 'Accept-Language: zh-CN,zh;q=0.9' 'https://www.baidu.com/s?wd=deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC&rn=10' -o /tmp/baidu_search.html; wc -c /tmp/baidu_search.html; python3 /tmp/baidu_parse.py /tmp/baidu_search.html
   [3/10] $ cat > /tmp/baidu_parse.py <<'PY'
import re, sys
from html import unescape
from html.parser import HTMLParser
from urllib.parse import unquote

path = sys.argv[1] if len(sys.argv) > 1 else "/tmp/baidu_search.html"
raw = open(path, encoding="utf-8", errors="ignore").read()


class _Text(HTMLParser):
    def __init__(self):
        super().__init__(convert_charrefs=True)
        self.parts = []

    def handle_data(self, data):
        self.parts.append(data)


def plain(text):
    text = re.sub(r"(?is)<(script|style).*?</\1>", " ", text)
    text = re.sub(r"<!--.*?-->", " ", text, flags=re.S)
    parser = _Text()
    parser.feed(text)
    return re.sub(r"\s+", " ", " ".join(parser.parts)).strip()


def real_url(block):
    match = re.search(r"(https?%3A%2F%2F[^\"&\\ ]+)", unescape(block))
    return unquote(match.group(1)) if match else ""


print(f"# 页面 {len(raw)} 字符")
for i, match in enumerate(re.finditer(r"<h3[^>]*>(.*?)</h3>", raw, re.S), 1):
    if i > 10:
        break
    block = match.group(1)
    title = plain(block)
    if not title:
        continue
    jump = re.search(r'href="(https?://[^"]+)"', block)
    print(f"{i}. {title}")
    if jump:
        print(f"   链接: {jump.group(1)}")
    if real := real_url(block):
        print(f"   真实地址: {real}")
    print(f"   摘要: {plain(raw[match.end(): match.end() + 2500])[:140]}")
PY
UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'; curl -sL --compressed -A "$UA" -H 'Referer: https://www.baidu.com/' -H 'Accept-Language: zh-CN,zh;q=0.9' 'https://www.baidu.com/s?wd=deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC&rn=10' -o /tmp/baidu_search.html; wc -c /tmp/baidu_search.html; python3 /tmp/baidu_parse.py /tmp/baidu_search.html
   ↳ 1488 /tmp/baidu_search.html # 页面 1438 字符

🤖 入门助手 > 命令:grep -o -e '百度安全验证' -e '网络不给力' -e 'result' /tmp/baidu_search.html | sort | uniq -c; head -c 600 /tmp/baidu_search.html
   [4/10] $ grep -o -e '百度安全验证' -e '网络不给力' -e 'result' /tmp/baidu_search.html | sort | uniq -c; head -c 600 /tmp/baidu_search.html
   ↳ 1 百度安全验证 1 网络不给力 <!DOCTYPE html> <html lang="zh-CN"> <head> <meta charset="utf-8"> <title>百度安全验证</title> <meta http-equiv="Content-Type" content="text/html; charset=utf-8"> <meta name="apple-mobile-web-app-capable" content="yes"> <meta name="apple-mobile-web-app-status-bar-style" content="black"> <m...

🤖 入门助手 > 命令:UA='Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1'; curl -sL --compressed -A "$UA" -H 'Accept-Language: zh-CN,zh;q=0.9' 'https://m.baidu.com/s?word=deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC' -o /tmp/baidu_m.html; wc -c /tmp/baidu_m.html; python3 /tmp/baidu_parse.py /tmp/baidu_m.html
   [5/10] $ UA='Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1'; curl -sL --compressed -A "$UA" -H 'Accept-Language: zh-CN,zh;q=0.9' 'https://m.baidu.com/s?word=deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC' -o /tmp/baidu_m.html; wc -c /tmp/baidu_m.html; python3 /tmp/baidu_parse.py /tmp/baidu_m.html
   ↳ 1853024 /tmp/baidu_m.html # 页面 1843099 字符 1. DeepSeek 简明解析,10分钟速通DeepSeekV1~V3核心技术点... 摘要: 2. Ollama实现本地部署 DeepSeek ! - 知乎 摘要: 操作系统：Windows、macOS 或 Linux。 下载 模型 ：：打开终端，输入以下命令下载并运行 DeepSeek 模型 。例如，下载 7B 版本 的命令为 大雄说技术： DeepSeek -R1持续霸榜！DeepSeek App榜单第一！ 知乎 3. 模型 & 价格 | DeepSeek API Docs 摘要: 4. DeepSee...

🤖 入门助手 > 命令:python3 - <<'PY'
import re
from html import unescape
raw = open('/tmp/baidu_m.html', encoding='utf-8', errors='ignore').read()
# 找含版本号关键词的片段
for kw in ['V3.2','V3.1','V3','R1','V4','最新版本','DeepSeek-V']:
    idxs = [m.start() for m in re.finditer(re.escape(kw), raw)][:3]
    for i in idxs:
        seg = re.sub(r'<[^>]+>',' ', raw[max(0,i-120):i+160])
        seg = re.sub(r'\s+',' ', unescape(seg)).strip()
        print(kw, '|', seg[:220])
    print('---')
PY
   [6/10] $ python3 - <<'PY'
import re
from html import unescape
raw = open('/tmp/baidu_m.html', encoding='utf-8', errors='ignore').read()
# 找含版本号关键词的片段
for kw in ['V3.2','V3.1','V3','R1','V4','最新版本','DeepSeek-V']:
    idxs = [m.start() for m in re.finditer(re.escape(kw), raw)][:3]
    for i in idxs:
        seg = re.sub(r'<[^>]+>',' ', raw[max(0,i-120):i+160])
        seg = re.sub(r'\s+',' ', unescape(seg)).strip()
        print(kw, '|', seg[:220])
    print('---')
PY
   ↳ --- --- V3 | JiELiQ7UUqF8+Yli0rR8XVjOvinlnEJKarWKHP9ufksrZweEO1vUTlYSR9Olc9pSHiUrLQxAU60O+kUsrtNPileXBaW5F9Uky2cuE1EO7Rxo1auRzSzFysmbV3LKpEQkKaBu1g40R5VWM/2CHHD48udMuKFSqsiSbboQ+0WvJQBUuSo0aDFAzUqXCVQLtqOeof5HmZWKMAFjwkb3OI6++QY9cyO8 V3 | 1fj/CsPk3kSukPxZe1JayLXaYWxbbTbusg1p0eu8jSz/t1P6hZjuBwCuZVLSa...

🤖 入门助手 > 命令:UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'; curl -sL --compressed -A "$UA" 'https://api-docs.deepseek.com/zh-cn/quick_start/pricing' -o /tmp/ds_pricing.html; wc -c /tmp/ds_pricing.html; python3 -c 'import html,re;t=open("/tmp/ds_pricing.html",encoding="utf-8",errors="ignore").read();t=re.sub(r"(?is)<(script|style).*?</\1>"," ",t);print(re.sub(r"\s+"," ",html.unescape(re.sub("<[^>]+>"," ",t)))[:1200])'
   [7/10] $ UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'; curl -sL --compressed -A "$UA" 'https://api-docs.deepseek.com/zh-cn/quick_start/pricing' -o /tmp/ds_pricing.html; wc -c /tmp/ds_pricing.html; python3 -c 'import html,re;t=open("/tmp/ds_pricing.html",encoding="utf-8",errors="ignore").read();t=re.sub(r"(?is)<(script|style).*?</\1>"," ",t);print(re.sub(r"\s+"," ",html.unescape(re.sub("<[^>]+>"," ",t)))[:1200])'
   ↳ 23313 /tmp/ds_pricing.html 模型 & 价格 | DeepSeek API Docs 跳到主要内容 DeepSeek API 文档 中文（中国） English 中文（中国） DeepSeek Platform 快速开始 首次调用 API 模型 & 价格 Token 用量计算 限速与隔离 错误码 接入 Agent 工具 API 指南 图像理解 思考模式 多轮对话 对话前缀续写（Beta） FIM 补全（Beta） JSON Output Tool Calls Files API 上下文硬盘缓存 使用 Responses API 使用 Anthropic API API ...

🤖 入门助手 > 命令:UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'; curl -sL --compressed -A "$UA" 'https://api-docs.deepseek.com/zh-cn/updates' -o /tmp/ds_upd.html; wc -c /tmp/ds_upd.html; python3 -c 'import html,re;t=open("/tmp/ds_upd.html",encoding="utf-8",errors="ignore").read();t=re.sub(r"(?is)<(script|style).*?</\1>"," ",t);print(re.sub(r"\s+"," ",html.unescape(re.sub("<[^>]+>"," ",t)))[:1000])'
   [8/10] $ UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'; curl -sL --compressed -A "$UA" 'https://api-docs.deepseek.com/zh-cn/updates' -o /tmp/ds_upd.html; wc -c /tmp/ds_upd.html; python3 -c 'import html,re;t=open("/tmp/ds_upd.html",encoding="utf-8",errors="ignore").read();t=re.sub(r"(?is)<(script|style).*?</\1>"," ",t);print(re.sub(r"\s+"," ",html.unescape(re.sub("<[^>]+>"," ",t)))[:1000])'
   ↳ 49093 /tmp/ds_upd.html 更新日志 | DeepSeek API Docs 跳到主要内容 DeepSeek API 文档 中文（中国） English 中文（中国） DeepSeek Platform 快速开始 首次调用 API 模型 & 价格 Token 用量计算 限速与隔离 错误码 接入 Agent 工具 API 指南 图像理解 思考模式 多轮对话 对话前缀续写（Beta） FIM 补全（Beta） JSON Output Tool Calls Files API 上下文硬盘缓存 使用 Responses API 使用 Anthropic API API 文档 新闻 其...

🤖 入门助手 > 完成:DeepSeek 最新模型版本（截至 2026-09-24，依据官方 API 文档与更新日志）：

**最新版本：DeepSeek-V4.1-Flash（2026-09-10 发布）**
- 官方定位：全新模型结构系列中最小尺寸的模型，原生多模态视觉理解能力
- API 模型名：`deepseek-flash`（旧名 `deepseek-v4-flash`、`deepseek-v4-flash-vision-exp` 已下线，请求会由 V4.1-Flash 承接）
- 上下文长度 1M，最大输出 384K，支持思考/非思考模式、Tool Calls、JSON Output、Responses API、Anthropic API
- 部分评测：GPQA Diamond 90.9、HLE 36.8、Codeforces 3471、Terminal-Bench 2.1 90.6

**同系列另一在售模型：DeepSeek-V4-Pro-0813**（模型名 `deepseek-v4-pro`，不支持图像理解）

**3 条来源链接**
1. 官方更新日志（V4.1-Flash 发布公告）：https://api-docs.deepseek.com/zh-cn/updates
2. 官方模型 & 价格页（模型版本、上下文、价格）：https://api-docs.deepseek.com/zh-cn/quick_start/pricing
3. 百度搜索结果页（第三方讨论，含 DeepSeek 模型/部署相关条目）：https://m.baidu.com/s?word=deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC

说明：百度网页版触发「百度安全验证」，改用移动端 m.baidu.com 抓取成功；核心版本信息以 DeepSeek 官方文档为准。

👤 你 > 
```

#### 下载视频

```text
👤 你 > 下载视频 https://www.bilibili.com/video/BV1ZLht69E7b ，然后看下 /tmp 是否有这个视频

🤖 入门助手 > 命令:ls agent/ && cat agent/skill.md 2>/dev/null | head -50
   [1/10] $ ls agent/ && cat agent/skill.md 2>/dev/null | head -50
   ↳ README.md __pycache__ agent.md config.py core.py llm.py main.py skill.md skills.py static tools.py web.py --- type: skill name: baidu_search description: "用 curl 访问百度网页搜索，抓取并解析搜索结果的标题/链接/摘要（纯命令行实现）" tags: [搜索, 百度, 网络, curl] --- # 技能：百度网页搜索（baidu_search） > 本文件是「技能文档」示例：Agent 通过 `cat agent/skill.md` 读...

🤖 入门助手 > 命令:which yt-dlp you-get lux ffmpeg 2>/dev/null; echo "---"; ls /tmp | head -30
   [2/10] $ which yt-dlp you-get lux ffmpeg 2>/dev/null; echo "---"; ls /tmp | head -30
   ↳ /Users/lucas/.penv/bin/yt-dlp --- roo-cline-idea-extension-ipc-1984918057065321528.sock roo-cline-idea-extension-ipc-7823245440825168466.sock

🤖 入门助手 > 命令:cd /tmp && yt-dlp -o "%(title)s.%(ext)s" "https://www.bilibili.com/video/BV1ZLht69E7b" 2>&1 | tail -20
   [3/10] $ cd /tmp && yt-dlp -o "%(title)s.%(ext)s" "https://www.bilibili.com/video/BV1ZLht69E7b" 2>&1 | tail -20
   ↳ [BiliBili] Extracting URL: https://www.bilibili.com/video/BV1ZLht69E7b [BiliBili] 1ZLht69E7b: Downloading webpage [BiliBili] BV1ZLht69E7b: Extracting videos in anthology [BiliBili] BV1ZLht69E7b: Downloading wbi sign [BiliBili] BV1ZLht69E7b: Downloading video formats for cid 42136046735 [BiliBili] Fo...

🤖 入门助手 > 命令:ls -lh /tmp/*.mp4 /tmp/*.m4a 2>/dev/null
   [4/10] $ ls -lh /tmp/*.mp4 /tmp/*.m4a 2>/dev/null
   ↳ -rw-r--r--@ 1 lucas wheel 65M Sep 24 02:26 /tmp/【闪客】这些 Jev 的案例都是骗人的！揭秘 Jev 玩我的世界效果骗局.f100026.mp4 -rw-r--r--@ 1 lucas wheel 4.9M Sep 24 02:26 /tmp/【闪客】这些 Jev 的案例都是骗人的！揭秘 Jev 玩我的世界效果骗局.f30280.m4a

🤖 入门助手 > 完成:视频已下载到 /tmp，但存在两个问题：

**下载结果**
- 视频文件：`/tmp/【闪客】这些 Jev 的案例都是骗人的！揭秘 Jev 玩我的世界效果骗局.f100026.mp4`（65 MB）
- 音频文件：`/tmp/【闪客】这些 Jev 的案例都是骗人的！揭秘 Jev 玩我的世界效果骗局.f30280.m4a`（4.9 MB）

**说明**
1. 视频标题为《【闪客】这些 Jev 的案例都是骗人的！揭秘 Jev 玩我的世界效果骗局》，BV 号 BV1ZLht69E7b。
2. 系统未安装 ffmpeg，因此视频流和音频流**没有合并**，是分开的两个文件（mp4 无声音，m4a 是音轨）。
3. 1080P 60帧 需要大会员，已自动降级到可用的清晰度（100026 格式）。
4. 如需合并成带声音的完整视频，需先安装 ffmpeg（如 `brew install ffmpeg`），再重新执行下载即可自动合并。
```

#### 清空临时目录

```text
👤 你 > 清空 /tmp 下的文件

🤖 入门助手 > 命令:rm -rf /tmp/* 2>/dev/null; ls -la /tmp | head -20
   [1/10] $ rm -rf /tmp/* 2>/dev/null; ls -la /tmp | head -20
   ↳ lrwxr-xr-x@ 1 root wheel 11 Sep 3 18:34 /tmp -> private/tmp

🤖 入门助手 > 命令:ls -la /private/tmp | head -30
   [2/10] $ ls -la /private/tmp | head -30
   ↳ total 16 drwxrwxrwt 3 root wheel 96 Sep 24 02:28 . drwxr-xr-x 6 root wheel 192 Sep 18 17:43 .. -rw-r--r--@ 1 lucas wheel 6148 Sep 24 02:25 .DS_Store

🤖 入门助手 > 完成:/tmp 下的文件已清空。

- 已删除之前下载的视频（65 MB mp4）和音频（4.9 MB m4a）等文件。
- 当前 `/private/tmp`（`/tmp` 是其符号链接）仅剩系统自动生成的 `.DS_Store`，无其他用户文件。
```

### 网页版

#### 启动记录

```text
/Users/lucas/.penv/bin/python /Users/lucas/PycharmProjects/ai/agent/web.py 

╭──────────────────────────────────────────────╮
│   📱 手机聊天网页（DeepSeek + 技能/命令）      │
╰──────────────────────────────────────────────╯
模型：deepseek-chat    人设：入门助手
技能：baidu_search
命令执行：开（与控制台版一致：会真的执行 shell 命令！）
本机访问：http://127.0.0.1:8000
手机访问（手机与电脑连同一个 Wi-Fi，然后在浏览器输入）：
   👉 http://192.168.1.4:8000
⚠️  注意：同一局域网内的设备都能访问本页并触发命令执行，用完请 Ctrl+C 关闭，或加 --no-tools
按 Ctrl+C 停止服务

[02:14:50] 192.168.1.4 "GET / HTTP/1.1" 200 -
[02:14:50] 192.168.1.4 "GET /static/style.css HTTP/1.1" 200 -
[02:14:50] 192.168.1.4 "GET /static/app.js HTTP/1.1" 200 -
[02:14:56] 192.168.1.4 "POST /api/chat HTTP/1.1" 200 -
[02:15:09] 192.168.1.4 "POST /api/chat HTTP/1.1" 200 -
[02:15:14] 192.168.1.4 "POST /api/chat HTTP/1.1" 200 -
[02:15:27] 192.168.1.4 "POST /api/chat HTTP/1.1" 200 -
[02:17:44] 192.168.1.4 "POST /api/chat HTTP/1.1" 200 -

👋 服务已停止

Process finished with exit code 0
```

网页截图

另外这个支持语音播报哦～

![](./image/agent/web-chat-agent.png)

---

## 3. 核心设计

### 3.1 为什么用「文本协议」而不是 function calling

| 维度   | function calling（首版方案）        | 文本协议（最终方案）                             |
| ---- | ----------------------------- | -------------------------------------- |
| 依赖   | 需要 SDK 或手写 tools 协议           | **零依赖**，纯文本                            |
| 可读性  | 工具调用藏在 JSON 里                 | 终端里一眼看到`命令:ls -la`                     |
| 工具定义 | 每个工具写 JSON Schema + Python 函数 | 就是「一条 shell 命令」                        |
| 灵活性  | 只能调注册过的函数                     | 任何命令行程序都能用（`curl`/`grep`/`python3 -c`） |
| 代价   | ——                            | 解析靠约定，需要人设反复强调格式；模型偶尔跑偏                |

结论：教学 demo 首选文本协议 —— 先把「循环 + 工具 + 上下文」这三件事看明白，之后再迁移到 function calling 只是换个 schema。

### 3.2 协议定义（写在 [`agent.md`](agent/agent.md:14)）

```text
你的目标是完成用户的任务，你必须选择下面的其中一种格式进行回复:

1.如果你认为需要执行命令，则输出'命令:XXX'，XXX 为命令本身，不要用任何的格式，不要解释

2，如果你认为不需要执行命令，则输出'完成:XXX'，XXX 为你的总结信息
```

`agent.md` 里还配套约束：一次只输出一条命令、命令在项目根目录执行、命令失败不要重复同一条、技能优先、结论必须来自真实命令输出、有预算意识。

### 3.3 协议解析：既要严格，也要容错

模型实际会输出各种变体，所以解析器做了三层容错（[`REPLY_RE`](agent/main.py:48) / [`_strip_code_fence()`](agent/main.py:52) / [`parse_reply()`](agent/main.py:65)）：

| 模型输出                          | 解析结果                                |
| ----------------------------- | ----------------------------------- |
| `命令:cat agent/skill.md`       | `("命令", "cat agent/skill.md")`      |
| `完成:字符数 = 1580`               | `("完成", "字符数 = 1580")`              |
| `# 命令:ls -a`                  | `("命令", "ls -a")`（容忍 `# > * \`` 前缀） |
| ```` ```\n命令:curl …\n``` ```` | `("命令", "curl …")`（剥掉一层代码块）         |
| 多行 heredoc 命令                 | 从标记一直取到结尾，保留换行                      |
| `我先看看有没有这个文件`                 | `None` → 打印「未按协议输出」，按完成处理           |

关键点：**载荷取到结尾**，因为命令可能是多行的（`python3 -c "…"`、`cat > /tmp/x.py <<'PY' …`）。

### 3.4 双层循环：这才是「Agent」

```text
外层（人机交互，main.py）
while True:
    user_input = input("👤 你 > ")
    messages.append(user)               # 用户输入进上下文
    内层（Agent 循环，run_turn）
    for step in 1..MAX_STEPS:
        reply = LLM(messages)           # ① 让模型决策（流式打印）
        messages.append(reply)          # ② 原文回填，保持协议一致
        kind, payload = parse(reply)    # ③ 解析协议
        if kind == "完成": break        # ④ 有结论 → 结束本轮
        output = os.popen(payload)      # ⑤ 执行命令（tools.py）
        messages.append(命令 + 执行结果) # ⑥ 观察结果回填 → 回到 ①
```

对应代码：[`run_turn()`](agent/main.py:104) 的 for 循环 + [`main()`](agent/main.py:192) 末尾的 `while True`。
把「模型回复原文」和「命令执行结果」都写回 `messages`，是让模型在多轮之间保持协议与记忆的关键 —— 这也是 **上下文工程（context engineering）** 的最小示例。

### 3.5 工具层：用 `os.popen` 执行命令

[`run_command()`](agent/tools.py:46) 做了四件事：

```python
# 1) 危险命令黑名单（[`BLOCKED_PATTERNS`](agent/tools.py:22) / [`_guard()`](agent/tools.py:33)）
#    命中即拒绝：sudo、rm -rf /、mkfs、dd of=/dev/*、fork bomb …
# 2) 固定工作目录 + 显式 shell + 引号转义，避免「命令在哪跑」「参数被拆」
full_command = f"cd {shlex.quote(str(WORKSPACE_DIR))} && {AGENT_SHELL} -c {shlex.quote(command)} 2>&1"
# 3) os.popen 只捕获 stdout，所以用 2>&1 把 stderr 合并进来（模型才看得到报错）
with os.popen(full_command) as pipe:
    output = pipe.read()
# 4) 输出截断（[`_truncate()`](agent/tools.py:40)），避免 900KB 网页把上下文撑爆
```

`os.popen` 的取舍（写进文件头注释，也是个知识点）：

- 优点：一行拿到命令输出，最适合教学；
- 缺点：**拿不到退出码、无法设置超时、没有沙箱** → 生产环境换 `subprocess.run(..., timeout=...)` + 容器/低权限用户/命令白名单。

### 3.6 观察结果怎么回填

```python
messages.append({
    "role": "user",
    "content": f"命令:{payload}\n\n命令执行结果:\n{output}\n\n(提示：本轮命令预算已用 {_step}/{MAX_STEPS} 条)",
})
```

见 [`main.py:139`](agent/main.py:139)。两个细节：

1. 用 `role="user"` 回填是最简做法（不需要改协议即可让模型看到观察结果）；
2. **把预算消耗写进观察结果**，模型会自发收敛（实测有效），比只在 system prompt 里说一次更有用。

### 3.7 预算控制与「强制收尾」

模型很容易陷入「再来一条命令确认一下」的循环。两层防护：

- 常规：[`run_turn()`](agent/main.py:104) 的 `for _step in range(1, MAX_STEPS + 1)`，默认 10 条；
- 兜底（[`main.py:146`](agent/main.py:146)）：预算用尽后，追加一条指令要求模型**立刻输出 `完成:`**，再给一次 LLM 调用机会。

实测对比（同一个问题「百度一下 deepseek 最新模型版本」）：

| 版本    | 结果                                      |
| ----- | --------------------------------------- |
| 强制收尾前 | 10 条命令用尽 → 直接打印「已达到上限」，**没有结论**，体验断掉    |
| 强制收尾后 | 10 条用尽 → 自动收尾，输出「结论 + 3 条来源链接 + 不确定项说明」 |

### 3.8 `agent.md`：Agent 的「角色卡」

front-matter 是机器配置，正文是给人看的（给模型看的）人设：

| 字段            | 作用                      | 谁在用                                         |
| ------------- | ----------------------- | ------------------------------------------- |
| `type: agent` | 与技能文档区分                 | 人工/AI 阅读                                    |
| `name`        | 终端显示的助手名                | [`main()`](agent/main.py:204)               |
| `model`       | 覆盖`DEEPSEEK_MODEL`      | [`main.py:210`](agent/main.py:210)          |
| `temperature` | 覆盖采样温度（跑偏就调 0）          | [`main.py:212`](agent/main.py:212)          |
| 正文            | system prompt 主体（含输出协议） | [`build_system_prompt()`](agent/main.py:77) |

> 好处：换人设、换模型、改协议都**不用动代码**。

### 3.9 `skill.md`：技能文档 = 命令步骤清单

技能文档的定位：**不写代码逻辑，只写「用哪些命令、按什么顺序、看什么输出、失败了怎么办」**。
因为在这个架构里，Agent 的全部能力就是「执行命令」。

[`skill.md`](agent/skill.md:1) 的结构（也是写新技能的模板）：

```text
front-matter: type/name/description/tags
## 何时使用            ← 让模型判断该不该加载
## 前置准备（只做一次）  ← date / curl --version / URL 编码关键词
## 步骤预算（很重要）    ← 最多 3 次搜索；信息够了就收尾；命令能合并就合并
## 执行步骤             ← 第 1 步落盘解析脚本 / 第 2 步 curl 抓取 / 第 3 步确认 / 第 4 步解析 …
## 被反爬拦住怎么办      ← 三级兜底：移动端 → 补 Referer → 换 Bing
## shell 引号避坑       ← 把「踩过的坑」写成规则
## 输出要求             ← 结论格式、必须带时间与来源、不得编造
```

配套机制是 **渐进式披露**：`skills.py` 只把技能的 `name/description/路径` 注入 system prompt（[`skills_overview()`](agent/skills.py:109)），
正文由模型自己 `cat agent/skill.md` 读进来（[`discover_skills()`](agent/skills.py:86)）。
好处：技能再多也不炸上下文，而且「要不要用这个技能」的决定权交给模型 —— 这正是 Agent 与固定 workflow 的区别。

---

### 3.10 网页版：手机聊天（[`web.py`](agent/web.py:1) + [`static/`](agent/static/index.html:1)）

需求：**一个聊天框，手机输入 URL 就能用，可选用语音把文字读出来。**

先把取舍写清楚，再写代码：

| 决策   | 选择                                        | 理由                                     |
| ---- | ----------------------------------------- | -------------------------------------- |
| 服务端  | 标准库`http.server.ThreadingHTTPServer`      | 与项目「零依赖」风格一致，`python3 agent/web.py` 即跑 |
| 传输   | SSE（`text/event-stream`）+ 前端 `fetch` 流式读取 | 打字机效果，不需要 WebSocket 的双向能力              |
| 前端   | 原生 HTML/CSS/JS 三个文件                       | 无构建步骤，改完刷新即可                           |
| 语音   | 浏览器原生`speechSynthesis`                    | 免 Key、免第三方服务，iOS/Android 都支持           |
| 上下文  | 内存字典 +`session_id`（localStorage 保存）       | 满足多轮对话，进程重启即清空，简单可控                    |
| 能力边界 | 网页版**不执行命令**                              | 手机端只聊天，风险面最小（执行命令只在控制台版）               |

请求链路：

```text
手机浏览器 POST /api/chat {session_id, message}
   → web.py 拼 messages（system 人设 + 最近 20 条历史 + 本轮输入）
   → llm.stream()               ← 与控制台版同一个 DeepSeek 客户端
   → 逐块写 SSE：data: {"delta": "…"}
   → app.js 用 ReadableStream 边收边渲染，最后 data: {"done": true}
   → 勾了「朗读回复」时：app.js 调 speechSynthesis 分句朗读（代码块/链接先剔除）
```

几个必须注意的细节（也是踩坑换来的）：

- SSE 响应头要带 `Cache-Control: no-cache`、`Connection: close`、`X-Accel-Buffering: no`，否则中间层会缓冲，打字机效果消失；
- `protocol_version = "HTTP/1.0"` 时响应结束即关连接，SSE 天然可用，不必手写 chunked 编码；
- 静态文件必须做**目录穿越防护**（`target.is_relative_to(STATIC_DIR)`），否则 `/../.env` 会把 API Key 读走；
- 只有真正拿到回答才写入历史，避免「一次失败的请求」污染后续上下文；
- 前端输入框字号设 16px，否则 iOS Safari 聚焦时会自动放大整个页面；
- 语音要**分句**朗读（长段落部分手机浏览器会截断），iOS 还需要一次用户手势来解锁播放（勾开关/点发送时顺手解锁）。

---

## 4. 开发过程实录（含真实踩坑与修复）

### 4.1 时间线

| 阶段      | 做了什么                                                                   | 结果                                 |
| ------- | ---------------------------------------------------------------------- | ---------------------------------- |
| ① 需求理解  | 4 条需求：目录 / 两个 md 示例 / while 交互 / DeepSeek                              | 先出「function calling 版」设计           |
| ② 首版实现  | `tools.py` 做函数注册表（JSON Schema），`llm.py` 支持 `tool_calls` 流式拼接           | **被否**：不符合「命令:/完成:」与 `os.popen` 要求 |
| ③ 方案重构  | 改为文本协议：`parse_reply()` 解析 + `os.popen` 执行；`llm.py` 去掉 function calling | 协议跑通                               |
| ④ 需求变更  | `skill.md` 从「文本统计」改为「百度网页查询」                                           | 技能文档重写为 curl 流程                    |
| ⑤ 离线自测  | 编译 + 协议解析 + 命令执行 + 技能发现 + prompt 组装                                    | 全部通过                               |
| ⑥ 端到端联调 | 真实 DeepSeek + 真实百度抓取（3 次）                                              | 暴露 4 个真实问题并修复                      |
| ⑦ 收尾    | 预算提示 + 强制收尾 + 文档（README / 本文）                                          | 稳定以`完成:` 结束                        |

### 4.2 踩坑清单（现象 → 原因 → 修法）

**坑 1：function calling 版不符合要求**

- 现象：工具调用走 JSON Schema，模型输出里没有 `命令:`/`完成:`
- 修法：整体改为文本协议，删掉 tools 注册表与 `tool_calls` 流式拼接；`llm.py` 只返回文本
- 收获：协议越简单，越容易调试；教学场景「所见即所得」比「协议完备」重要

**坑 2：命令报错模型看不到**

- 现象：`cat /tmp/not-exist` 只返回「无输出」，模型不知道该改路径
- 原因：`os.popen` 只捕获 stdout
- 修法：命令前后套 `bash -c ... 2>&1`，把 stderr 合并（[`tools.py:63`](agent/tools.py:46)）
- 验证：`cat /tmp/definitely-not-exist-abc` → `cat: /tmp/…: No such file or directory` 正常回填

**坑 3：命令到底在哪个目录执行？**

- 现象：模型写 `cat agent/skill.md`，如果 cwd 不对就全盘皆错
- 修法：`cd {WORKSPACE_DIR} && bash -c '<cmd>'`，并在 system prompt 里写明「工作目录=项目根目录」；`main()` 里也 `os.chdir(WORKSPACE_DIR)`（[`main.py:198`](agent/main.py:198)）
- 附带：`shlex.quote(command)` 防止带空格/引号的命令被 shell 拆碎

**坑 4：shell 嵌套引号，命令直接语法错误**

- 现象：模型写了
  `python3 -c "… re.findall(r'href="(https?://…)"', t) …"`
  → shell 把内层 `"` 当成字符串结束，Python 收到半截代码，`SyntaxError`
- 修法（写进 [`skill.md` 的「shell 引号避坑」](agent/skill.md:145)）：
  1. `python3 -c` 的代码**用单引号包裹**，代码内部一律用双引号；
  2. 超过一行、或含 `$`/反斜杠/引号嵌套 → **先用 heredoc 落盘 `/tmp/xxx.py` 再执行**
- 验证：解析脚本用 heredoc 落盘后，一次执行成功输出标题 + 链接 + 摘要

**坑 5：正则剥 HTML 标签会残留碎片**

- 现象：摘要里出现 `  <d`、`0 cos-text-body false false…` 之类垃圾
- 原因：`<[^>]+>` 遇到**属性值里含 `>`**（百度 `data-click` 的 JSON 就有）时会错位切割
- 修法：改用标准库 `HTMLParser` + `convert_charrefs=True` 提取文本（[`skill.md` 第 1 步](agent/skill.md:37)）
- 收获：能不用正则解析 HTML 就别用

**坑 6：卡片类结果的摘要为空**

- 现象：百度百科/官网卡片由 JS 渲染，解析出的摘要为空，模型于是反复重抓同一页
- 修法：① 技能文档明确写「摘要经常为空，这是正常的，不要重抓」；② 允许「标题 + 真实地址」作为有效来源；③ 在「步骤预算」里规定最多 3 次搜索

**坑 7：预算耗尽却没有结论**

- 现象：10 条命令用尽，终端只打印「已达上限」，用户拿不到答案
- 修法：预算耗尽 → 自动追加「立刻输出'完成:'」→ 再给一次调用；同时每次回填观察结果都带上「预算已用 n/10」
- 位置：[`main.py:139`](agent/main.py:139) 与 [`main.py:146`](agent/main.py:146)

**坑 8：桌面端百度被反爬拦截**

- 现象：`curl www.baidu.com/s?wd=…` 只返回 1488 字节（安全验证页）
- 修法：技能文档给出三级兜底，实测**移动端 `m.baidu.com` 成功**（抓到 185 万字符页面，解析出标题与链接）
- 收获：把「失败分支」写进技能文档，Agent 才能自己从坑里爬出来

**坑 9：危险命令防护的验证**

- 现象：需要确认 `os.popen` 不是「裸奔」
- 验证：模型/手工执行 `sudo ls` → 返回 `[已拦截] 命令命中危险模式 …`（[`BLOCKED_PATTERNS`](agent/tools.py:22)）
- 说明：这只是「粗护栏」，真正的安全边界是容器/低权限账户

---

**坑 10：静态资源 404（`/static/` 前缀没剥干净）**

- 现象：`GET /` 正常，但 `GET /static/app.js` 返回 404
- 原因：`STATIC_DIR` 已经是 `agent/static`，代码又拼了一次 `static/app.js`
- 修法：先剥掉 URL 里的 `/static/` 前缀再拼路径（[`_serve_static()`](agent/web.py:140)），并顺手支持 `/style.css` 这种直接访问写法
- 收获：静态服务这种事最容易「想当然」，写完就 curl 一遍

**坑 11：共享代码的目录归属（返工两次）**

- 第一次：把 DeepSeek 客户端抽到项目根目录的独立共享包，两个入口分别 import
- 反馈：**「这就是一个 agent 开发的 demo，代码应该放在一起，LLM 请求用同一个」**
- 最终：所有 Python 回到 `agent/`；`llm.py` 是唯一的 DeepSeek 客户端（`chat()` 给控制台、`stream()` 给网页），
  配置只有一份 [`config.py`](agent/config.py:1)，凭据只有一份 `agent/.env`
- 收获：**目录结构要服从使用者的心智模型**；「复用」不等于「抽成独立包」——同目录内共用一份实现更简单，
  还省掉了 `sys.path` 注入这类隐藏依赖

---

## 5. 联调与验证记录

### 5.1 离线自测（不花钱，先保证机械部分正确）

```bash
python3 -m py_compile agent/*.py          # 编译检查
python3 - <<'PY'                          # 协议解析 / 命令执行 / 技能发现 / prompt 组装
import sys; sys.path.insert(0, "agent")
from main import parse_reply, build_system_prompt
from skills import discover_skills, load_markdown
from tools import run_command
...
PY
```

实测输出（节选）：

```text
== 协议解析 ==
'命令:cat agent/skill.md' -> ('命令', 'cat agent/skill.md')
'完成:字符数 = 1580'      -> ('完成', '字符数 = 1580')
'# 命令:ls -a'            -> ('命令', 'ls -a')
'我先看看有没有这个文件'   -> None

== 技能发现 ==
baidu_search | 用 curl 访问百度网页搜索… | agent/skill.md

== 命令执行 (os.popen) ==
2                                      # python3 -c 'print(1+1)'
[已拦截] 命令命中危险模式 `\bsudo\b`    # sudo ls
```

### 5.2 端到端联调（真实 API + 真实抓取）

```bash
AGENT_MAX_STEPS=3 python3 agent/main.py -p "百度一下 deepseek 最新模型版本"   # 验证强制收尾
python3 agent/main.py -p "百度一下 deepseek 最新模型版本，给出 3 条来源链接"   # 默认预算 10
python3 agent/main.py -p "agent 目录里有几个 python 文件？各多少行？"        # 本地任务
```

真实轨迹（节选，能看清 Agent 的「思考-行动」过程）：

```text
👤 你 > 百度一下 deepseek 最新模型版本，给出 3 条来源链接

🤖 入门助手 >
   [1/10] $ cat agent/skill.md                       ← 先读技能文档
   ↳ --- type: skill name: baidu_search …
   [2/10] $ date '+%Y-%m-%d %H:%M'; python3 -c '…quote(sys.argv[1])' 'deepseek 最新模型版本'
   ↳ 2026-09-24 01:18  deepseek%20%E6%9C%80%E6%96%B0%E6%A8%A1%E5%9E%8B%E7%89%88%E6%9C%AC
   [3/10] $ cat > /tmp/baidu_parse.py <<'PY' … PY     ← 按技能文档落盘解析脚本
   [4/10] $ UA='Mozilla/5.0 …'; curl -sL --compressed -A "$UA" 'https://www.baidu.com/s?wd=…&rn=10' -o /tmp/baidu_search.html; wc -c …
   ↳ 1488 /tmp/baidu_search.html                     ← 桌面端被反爬拦截
   …                                                 ← 模型自行切到移动端兜底
   完成:本次通过百度移动端搜索（2026-09-24 01:18）… 3 条来源链接 … 不确定项 …
```

结论：

- 机械链路（协议解析 → 命令执行 → 结果回填 → 循环收敛）**完全可用**；
- 「信息是否足够」的判断仍依赖模型，靠技能文档 + 预算约束 + 强制收尾来收敛；
- 百度抓取天然不稳定，**技能文档里的兜底与「不许编造」约束是必要的**。

---

### 5.3 网页版联调（真实 DeepSeek 流式 + 静态资源 + 多轮上下文）

```bash
python3 agent/web.py --port 8125      # 启动，日志里会打印手机可访问的局域网地址
curl -sN -X POST http://127.0.0.1:8125/api/chat \
     -H 'Content-Type: application/json' \
     -d '{"session_id":"ctx","message":"用一句话介绍你自己"}'
```

实测结果（节选）：

```text
GET /                 → 200  1100B  text/html; charset=utf-8
GET /static/app.js    → 200  6776B  text/javascript; charset=utf-8
GET /static/style.css → 200  3620B  text/css; charset=utf-8
GET /../config.py     → 404       ← 目录穿越被拒绝

SSE（时间戳相同但确实是逐块到达，说明是流式而不是一次性返回）：
01:49:24  data: {"delta": "我是一个"}
01:49:24  data: {"delta": "简洁"}
01:49:24  data: {"delta": "直接的"}
…
data: {"done": true}

多轮上下文（服务端按 session_id 记忆）：
[第1轮] 我最喜欢的数字是 7，请只回复“好的”   → 好的
[第2轮] 我喜欢几？只回答数字                  → 7
```

结论：网页版链路（SSE 推送 → 前端逐块渲染 → 服务端历史 → 朗读开关）全部可用；
手机端只要和电脑在同一 Wi-Fi、防火墙放行，直接访问启动日志里的 `http://192.168.x.x:端口` 即可聊天。

---

## 6. 逐文件代码导读

| 文件                                       | 关键对象                                        | 行号      | 作用                                        |
| ---------------------------------------- | ------------------------------------------- | ------- | ----------------------------------------- |
| [`config.py`](agent/config.py:15)        | `WORKSPACE_DIR`                             | 15      | 命令执行 / 文件访问的根目录                           |
|                                          | [`load_dotenv()`](agent/config.py:22)       | 22      | 极简 .env 解析（两个入口共用）                        |
|                                          | `MAX_STEPS` / `AGENT_SHELL`                 | 56 / 58 | 控制台版：预算与 shell                            |
|                                          | `WEB_PORT` / `WEB_SYSTEM_PROMPT`            | 62 / 66 | 网页版：端口与人设                                 |
| [`llm.py`](agent/llm.py:33)              | [`DeepSeekLLM`](agent/llm.py:33)            | 33      | ⭐ 两个入口共用的 OpenAI 兼容客户端                    |
|                                          | [`_iter_pieces()`](agent/llm.py:85)         | 85      | SSE 增量解析（正文 + reasoner 思维链）               |
|                                          | [`stream()`](agent/llm.py:110)              | 110     | 逐块产出正文 → 网页版 SSE 用                        |
|                                          | [`complete()`](agent/llm.py:117)            | 117     | 一次性拿完整回答                                  |
|                                          | [`chat()`](agent/llm.py:126)                | 126     | 边收边打印 → 控制台版用                             |
| [`tools.py`](agent/tools.py:46)          | [`run_command()`](agent/tools.py:46)        | 46      | `os.popen` 执行命令（cd + bash -c + 2>&1 + 截断） |
|                                          | [`_guard()`](agent/tools.py:33)             | 33      | 危险命令黑名单                                   |
| [`skills.py`](agent/skills.py:47)        | [`parse_markdown()`](agent/skills.py:47)    | 47      | 拆 front-matter（`key: value` / `[a, b]`）   |
|                                          | [`discover_skills()`](agent/skills.py:86)   | 86      | 扫描`*.md` + `skills/*.md`，收 `type: skill`  |
|                                          | [`skills_overview()`](agent/skills.py:109)  | 109     | 生成注入 prompt 的技能清单（表格）                     |
| [`web.py`](agent/web.py:63)              | [`ChatHandler`](agent/web.py:63)            | 63      | 路由：静态文件 +`POST /api/chat`                 |
|                                          | [`_start_sse()`](agent/web.py:102)          | 102     | 开 SSE（禁缓存 + 关连接 + 禁反代缓冲）                  |
|                                          | [`_serve_static()`](agent/web.py:140)       | 140     | 静态文件 + 目录穿越防护                             |
|                                          | [`_handle_chat()`](agent/web.py:156)        | 156     | 拼 messages → 流式回写 + 记历史                   |
| [`static/app.js`](agent/static/app.js:1) | `send()`                                    | 1       | fetch + ReadableStream 边收边渲染；`speak()` 朗读 |
| [`main.py`](agent/main.py:65)            | [`parse_reply()`](agent/main.py:65)         | 65      | 解析`命令:` / `完成:`（含容错）                      |
|                                          | [`build_system_prompt()`](agent/main.py:77) | 77      | 正文 + 技能清单 + 运行环境 + 协议强调                   |
|                                          | [`run_turn()`](agent/main.py:104)           | 104     | **Agent 循环**（含预算提示与强制收尾）                  |
|                                          | [`handle_command()`](agent/main.py:172)     | 172     | 本地`/help` `/skills` `/clear` `/exit`      |
|                                          | [`main()`](agent/main.py:192)               | 192     | 装配 + 外层`while input()`                    |

---

## 7. 二次开发指南

### 7.1 加一个技能（不改代码）

在 `agent/skills/xxx.md`（子目录会被扫描）写：

```markdown
---
type: skill
name: my_skill
description: "一句话说明能做什么（会注入 system prompt）"
tags: [demo]
---

# 技能：xxx
## 何时使用
## 前置准备
## 步骤预算
## 执行步骤（一条命令一步，写清用什么命令、看什么输出）
## 出错了怎么办
## 输出要求
```

要点：把**失败分支**和**引号/路径注意事项**写进去 —— 技能文档的质量直接决定 Agent 的成功率。

### 7.2 换模型 / 换人设 / 调稳定性

```bash
DEEPSEEK_MODEL=deepseek-reasoner python3 agent/main.py   # 看 🧠 思考过程
DEEPSEEK_TEMPERATURE=0 python3 agent/main.py             # 协议跑偏时把温度调 0
AGENT_MAX_STEPS=15 python3 agent/main.py                 # 复杂任务放宽预算
AGENT_COMMAND_OUTPUT_MAX_CHARS=8000 python3 agent/main.py # 摘要类任务放宽输出
```

或在 [`agent.md`](agent/agent.md:1) 的 front-matter 里改 `model` / `temperature`（覆盖环境变量）。

### 7.3 加「工具」而不是「裸命令」

当前所有能力都靠 shell。想收窄权限，可以在 [`tools.py`](agent/tools.py:46) 里加一层白名单：
只允许 `cat/ls/grep/curl/python3 …` 前缀，其它命令返回 `[已拦截]`，再把允许清单同步写进 `agent.md`。

### 7.4 迁移到 function calling / MCP

文本协议熟练后，迁移只要三步（[`llm.py`](agent/llm.py:73) 是唯一需要改协议的地方）：
① `chat()` 的 payload 加回 `tools`；② 流式解析里按 `index` 拼接 `tool_calls`（首版代码里已实现过）；
③ `run_turn()` 把「解析文本」换成「遍历 tool_calls」。

### 7.5 更进一步

- **记忆**：`messages` 落盘（JSON/SQLite）+ 长历史摘要压缩；
- **可观测**：记录每条命令、耗时、token 用量，评估成功率；
- **多 Agent**：`agent.md` 当角色卡，研究员 + 撰写 + 审校各配技能目录；
- **更稳的搜索**：把 Google/Bing/Brave API 或 `duckduckgo-search` 包成技能，替掉脆弱的 HTML 解析。

---

## 8. 安全与局限

| 项     | 现状                                         | 生产建议                               |
| ----- | ------------------------------------------ | ---------------------------------- |
| 命令执行  | `os.popen`，能跑任意 shell                      | `subprocess` + 超时 + 白名单；容器/低权限账户隔离 |
| 超时    | 无（长命令会挂住）                                  | 必须加超时（`timeout` 参数）                |
| 黑名单   | 7 条粗规则（[`tools.py:22`](agent/tools.py:22)） | 黑名单永远不完备，改用白名单                     |
| 网络    | 会带 UA/Referer 抓百度                          | 遵守 robots.txt 与站点条款，别高频抓取          |
| 密钥    | `agent/.env`                               | 勿提交仓库；用环境变量或密钥管理服务                 |
| 结论可靠性 | 依赖模型判断，可能给不出确定结论                           | 强制要求「来源 + 抓取时间 + 不确定项」，可加二次校验      |

---

## 9. 排错速查

| 现象                      | 原因 / 处理                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `未找到 DEEPSEEK_API_KEY`  | `cp agent/.env.example agent/.env` 填 Key，或 `export DEEPSEEK_API_KEY=sk-xxx` |
| `HTTP 401 / 402 / 429`  | Key 无效 / 余额不足 / 限流                                                          |
| 模型不输出`命令:` `完成:`        | 把`DEEPSEEK_TEMPERATURE` 调 0；确认 `agent.md` 的协议段落没被改坏                         |
| 命令报`SyntaxError` / 引号错误 | 按[`skill.md` 的避开章节](agent/skill.md:145)：单引号包 `python3 -c`，复杂代码 heredoc 落盘   |
| 命令报「command not found」  | 模型用了不存在的命令 → 人设里加「先`ls`/`command -v` 确认」                                    |
| 抓百度返回 1KB 左右            | 被安全验证拦截 → 移动端`m.baidu.com` 或换 Bing 源                                        |
| 命令数用尽没结论                | 已内置强制收尾；实在不够就调大`AGENT_MAX_STEPS`                                            |
| 摘要一直为空                  | 卡片类结果 JS 渲染，属正常；用标题 + 真实地址即可                                                |

---

## 10. 小结：这个 demo 教会我们什么

1. **Agent = LLM + 循环 + 工具**：`run_turn()` 的 for 循环就是全部魔法，别的都是工程细节。
2. **协议越简单越好调试**：`命令:` / `完成:` 两个前缀，让「模型决策」变得肉眼可见。
3. **上下文工程是核心**：模型回复原文、命令执行结果、预算提示，都要回填，模型才能连贯地干活。
4. **预算必须有**：条数上限 + 观察结果里的预算提示 + 耗尽后强制收尾，缺一个都会出现「转圈不结账」。
5. **技能文档是「外挂的流程知识」**：把步骤、失败兜底、引号坑写成 markdown，Agent 就能跨出模型自身的知识边界。
6. **能力越强越要设边界**：`os.popen` 让 Agent 真的能动手，所以黑名单、工作目录限制、输出截断一个都不能少。
