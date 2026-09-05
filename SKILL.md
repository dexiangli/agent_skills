---
name: ima-weekly-review
display_name: "ima 知识库周复盘自动化"
description: "从 ima 知识库扫描上周笔记，生成完整周复盘报告，存档到知识库，并通过 PushPlus 推送微信精简版"
version: "1.0"
agent_created: true
triggers:
  - 周复盘
  - 知识库周总结
  - weekly review
  - ima 知识库复盘
---

# ima 知识库周复盘自动化

## 适用场景
每周定时从 ima 知识库扫描上周新增/更新的笔记，生成完整复盘报告（报告A：上周总结 + 报告B：状态对比），存档到知识库指定文件夹，并通过 PushPlus 推送微信精简版。

## 前置条件
- ima 知识库 MCP 工具已连接（`mcp__ima-mcp__*`）
- PushPlus token 已配置

## 配置参数

> **首次使用必读**：下面所有 `<...>` 都是占位符。
> 用 `get_knowledge_base_list` 拿到你的知识库 ID，用 `get_knowledge_list` 逐级拿到各文件夹 ID，
> PushPlus token 在 pushplus.plus 登录后免费获取。填进下表后本 skill 才能跑。
> **千万别把真实 token 提交进公开仓库。**


| 参数 | 值 |
|------|-----|
| 知识库 ID | `<YOUR_KB_ID>` |
| 库名 | <你的知识库名称> |
| 周复盘存档文件夹 ID | `<FOLDER_ID_周复盘存档>` |
| PushPlus token | `<YOUR_PUSHPLUS_TOKEN>` |

## 重点文件夹 ID

| 文件夹 | folder_id |
|--------|-----------|
| 收集箱 | `<FOLDER_ID_收集箱>` |
| 💭 深度思考 | `<FOLDER_ID_深度思考>` |
| 🚀 当前任务 | `<FOLDER_ID_当前任务>` |
| 📚 参考资源 | `<FOLDER_ID_参考资源>` |
| 📦 历史箱（原"搁置区"） | `<FOLDER_ID_历史箱>`，内含子文件夹：📁 历史原件(`<FOLDER_ID_历史原件>`，其下按「2026年8月」→「08-02」逐日归档)、📦 错误件(`<FOLDER_ID_错误件>`)、灵感种子(`<FOLDER_ID_灵感种子>`)、📦 临时中转(`<FOLDER_ID_临时中转>`)、📦 沉淀箱(`<FOLDER_ID_沉淀箱>`) |
| 🔍 痛点追踪 | `<FOLDER_ID_痛点追踪>`（含：情绪管理、拖延、装聪明心态、自我认知模糊 等） |

> 注意：日记类记录日常由 AI 整理归档到 历史原件/2026年8月/08-XX 子文件夹（media_id 与收集箱一致，不用重复读正文）。窗口内日期=标题前缀（如「8.8 利用旧电脑」记录 8.8 内容但可能 8.9 晚才入库，create_time 在窗口外——**按标题日期归属，不要按 create_time 一刀切**）。
| 🗄️ 搁置区 | `<FOLDER_ID_历史箱>` |

## 完整执行步骤

### Step 0: 补检兜底（本地 + 知识库双校验）

**必须两处都查**，任一命中就直接结束、不重复生成也不重复推送：

1. **本地**：工作目录下是否已有 `周复盘_<本次区间>.md`
2. **知识库**：
```
get_knowledge_list(knowledge_base_id="<YOUR_KB_ID>", folder_id="<FOLDER_ID_周复盘存档>",
                   limit=50, cursor="", sort_type="UPDATE_TS_DESC_SORT_TYPE")
```

> ⚠️ 为什么要双校验：`add_knowledge` 成功后 `get_knowledge_list` 可能因缓存延迟返回空。只查知识库会误判成"还没生成"，导致重复生成 + 重复推送微信。本地文件是可靠的第二道锁，所以 Step 6 存档时**必须先写本地再传云端**。

两处都无 → 继续执行。这样错过定时触发后手动补跑，也能自动补上漏掉的那周。

### Step 1: 确定日期窗口
区间 = `[运行日-7天, 运行日-1天]`，共 7 个自然日，含头含尾。周日 20:00 运行 → 覆盖上周日至本周六。
先用 Bash `date` 取真实当前日期，**不要凭空推算**。
文件名格式：`周复盘_YYYY-MM-DD_YYYY-MM-DD.md`

> ⚠️ **补跑窗口判定（2026-08-17 实测）**：错过周日定时、周一或之后才补跑时，不要机械套 `[运行日-7, 运行日-1]`（否则会把上周六/周日算进新窗口、漏掉上周日）。先看「周复盘存档」里最新报告的截止日期，窗口取 **`[上次报告截止日+1, 最近一个完整周的周六]`**，与上一份无缝衔接。例：上次报告截止 08-08，8/17 补跑 → 窗口 [08-09, 08-15]，而非 [08-10, 08-16]。

### Step 2: 扫描上周内容（直达归档文件夹，不遍历全库）

**⛔ 最重要的一条：必须用 `fetch_media_content(media_id=...)` 读正文全文。**

`get_knowledge_list` 返回的 `introduction` 是 ima 自动生成的 AI 摘要，**不是用户写的原话**。在摘要上再做一次 AI 加工 = 两层抽象叠加 = 产出全是"正确的废话"，用户会直接反馈"水分太大、看不到真实痛点"。
`introduction` 只能用来做窗口筛选和判断要不要读，**绝不能拿它当写报告的素材**。

**快捷扫描路径（2026-08-09 优化，已验证）：** 日记日常已按「📁 历史原件 › 2026年8月 › 08-XX」按日归档，只取对应文件夹，不要翻收集箱全量、不要遍历整库：

1. **窗口内日记（主渠道）**：
   a) 列「历史原件 › 2026年8月」`get_knowledge_list(knowledge_base_id=..., folder_id="<FOLDER_ID_月份归档>", limit=50, cursor="", sort_type="UPDATE_TS_DESC_SORT_TYPE")` → 返回 08-XX 日文件夹
   b) 对窗口内的每个 08-XX 子文件夹（folder_id 用返回的 `folder_info.folder_id`），再 `get_knowledge_list(..., limit=50, ...)` 取当日全部条目
2. **主题笔记**：直接列以下文件夹（各一次 `get_knowledge_list`，limit=50，UPDATE_TS_DESC）：
   - 💭 深度思考 `<FOLDER_ID_深度思考>`
   - 🚀 当前任务 `<FOLDER_ID_当前任务>`
   - 🔍 痛点追踪 `<FOLDER_ID_痛点追踪>`
   - 📚 参考资源 `<FOLDER_ID_参考资源>`（通常无新增，无则跳过）
3. **兜底**：某日文件夹缺失/条目疑似不全，或某 folder_id 报"文件夹不存在"时，用 `search_knowledge(knowledge_base_id=..., query=<主题词>)` 搜索补漏，再按时间筛。

调用方式（注意 `limit` 是**必填**参数且**上限 50**——写成 100 会报 `invalid GetKnowledgeListReq.Limit: value must be inside range (0, 50]`，2026-08-09 实测）：

```
get_knowledge_list(
  knowledge_base_id="<YOUR_KB_ID>",
  folder_id="folder_xxx",
  limit=50,
  cursor="",
  sort_type="UPDATE_TS_DESC_SORT_TYPE"
)
```

- `sort_type="UPDATE_TS_DESC_SORT_TYPE"` 按更新时间倒序，遇到早于窗口起点的条目即可停止该文件夹，不必全量遍历
- 条目超过 `limit` 时用返回的 `cursor` 翻页
- 返回项若是 `FOLDER` 类型（月份子文件夹），递归进去再列一层
- 需要时可用 `filters` 按 `media_type` 过滤（NOTE / MARKDOWN / WEB 等）
- **日记按标题日期归属窗口**（如「8.8 利用旧电脑」可能 8.9 晚才入库、create_time 在窗口外）——按标题日期判断，不按 create_time 一刀切，避免漏掉晚入库的窗口内容

> ⚠️ 注意：知识库文件夹 ID 可能会随重建而变化，搜索方式更可靠。

### Step 3: 建素材清单

读完正文后，先建一份内部素材清单（不进最终报告）：每条记 **标题 / 日期 / 所在文件夹 / 用户自己写下的原句**（尤其是带判断、疑问、情绪、待办的句子）。

报告里的每个结论都必须能追溯到这份清单——写不出出处的结论一律删掉。这是反水分的机制保障。

### Step 4: 读取历史基准
从「周复盘存档」取日期区间早于本周的最新一份报告，同样用 `fetch_media_content` 读**正文**作对比基准。
若无历史报告，注明"首次运行，暂无对比基准"，跳过报告 B 的对比，只保留卡点与建议。

### Step 5: 生成完整报告

**反水分是第一优先级。** 硬性禁令：

- 不写"这体现了你在 XX 方面的成长/思维升级"这类拔高句
- 不写笔记里没有的推论，只陈述笔记里明确写过的事实
- 不用"深度思考""认知跃迁""系统性提升"这类空词
- 不为填满模板而扩写；某板块没料就直接写"本周无"
- 有效笔记 ≤ 3 条时如实说"本周输入很少"，报告就短，不拉长窗口凑数

**报告 A：上周实况**
- 本周实际做了什么（按主题归纳，标注来源笔记标题）
- 想通了什么（严格标准：笔记里出现"原来 / 我发现 / 想明白了 / 关键是"才算，找不到就写"无明确认知突破记录"，不许凑数量）
- 进行中任务的实际进展（有进度数字写数字，没有写"未记录具体进展"）
- **卡点与未闭环（重点板块）**：明确写出的困惑、反复出现在多篇笔记里的同一问题（标注出现次数与篇目）、开了头没写完/有问号没答案/待办没打勾的条目

**报告 B：状态对比（证据驱动）**
五维度标 提升 / 持平 / 下滑 / **数据不足**：知识产出量、认知深度、行动闭环、任务推进、状态/精力。

- 每个维度必须附具体证据（本周 X 条 vs 上周 Y 条、具体哪几篇）
- 给不出证据一律写"数据不足，不判断"，禁止凭感觉贴标签
- 「状态/精力」只在笔记里有直接表述（累了/没时间/焦虑等原话）时才判断，**禁止从行文语气脑补**

**建议**：2-4 条，只针对报告 A 列出的具体卡点，每条必须是"下周能直接做的一个动作"，不是"要加强 XX 意识"。不强制套库内方法论，套不上就别扯。

### Step 6: 存档到知识库
0. **先把完整报告写到本地工作目录**（这是下次 Step 0 防重复的依据，必须先写成功）
1. `create_media(knowledge_base_id, file_name, file_size, content_type="text/markdown")` 获取 media_id + COS 凭证
2. 上传到 COS（推荐用 `cos-python-sdk-v5`，详见下方；或标准库签名上传）
3. `add_knowledge(knowledge_base_id, media_id, folder_id)` 入库

> ⚠️ 注意：add_knowledge 返回成功但 get_knowledge_list 可能仍显示空，是 API 缓存延迟问题，不影响实际存储。

### Step 7: 推送微信精简版
**⚠️ 关键：必须用文件方式推送，不能内联中文 JSON，否则 Git Bash 会乱码！**

## COS 上传（推荐方案：cos-python-sdk-v5）

手动签名容易踩坑，推荐用腾讯云 COS SDK（需先 `pip install cos-python-sdk-v5`）：

```python
from qcloud_cos import CosConfig, CosS3Client

config = CosConfig(
    SecretId='<secret_id>',
    SecretKey='<secret_key>',
    Token='<token>',
    Region='<region>',
    Scheme='https'
)
client = CosS3Client(config)
response = client.put_object(
    Bucket='<bucket>',
    Body=open(local_file_path, 'rb'),
    Key='<cos_key>',
    ContentType='text/markdown'
)
```

## COS 上传签名算法（备选：标准库实现）

COS 签名格式容易出错，以下是验证通过的 Python 标准库实现：

```python
import hashlib, hmac, time, urllib.request, urllib.parse

# 从 create_media 返回值获取
secret_id = '<cos_credential.secret_id>'
secret_key = '<cos_credential.secret_key>'
token = '<cos_credential.token>'
bucket = '<cos_credential.bucket_name>'
region = '<cos_credential.region>'
cos_key = '<cos_credential.cos_key>'  # 含前导 /

# 读取文件
with open(file_path, 'rb') as f:
    content = f.read()

host = f'{bucket}.cos.{region}.myqcloud.com'
now = int(time.time())
key_time = f'{now};{now+3600}'

# SignKey
sign_key = hmac.new(secret_key.encode(), key_time.encode(), hashlib.sha1).hexdigest()

# HttpString - headers 必须按字母排序，key 小写，value URL-encode，用 & 连接
headers_lower = {
    'content-length': str(len(content)),
    'content-type': 'text/markdown',
    'host': host
}
sorted_keys = sorted(headers_lower.keys())
header_list = ';'.join(sorted_keys)
header_parts = [f'{k}={urllib.parse.quote(headers_lower[k], safe="")}' for k in sorted_keys]
header_kv = '&'.join(header_parts)

http_string = f'put\n{cos_key}\n\n{header_kv}\n'

# StringToSign
sha1_http_string = hashlib.sha1(http_string.encode()).hexdigest()
string_to_sign = f'sha1\n{key_time}\n{sha1_http_string}\n'

# Signature
signature = hmac.new(sign_key.encode(), string_to_sign.encode(), hashlib.sha1).hexdigest()

# Authorization
auth_header = f'q-sign-algorithm=sha1&q-ak={secret_id}&q-sign-time={key_time}&q-key-time={key_time}&q-header-list={header_list}&q-url-param-list=&q-signature={signature}'

# Upload
url = f'https://{host}{cos_key}'
req = urllib.request.Request(url, data=content, method='PUT')
req.add_header('Host', host)
req.add_header('Content-Type', 'text/markdown')
req.add_header('Content-Length', str(len(content)))
req.add_header('Authorization', auth_header)
req.add_header('x-cos-security-token', token)
resp = urllib.request.urlopen(req, timeout=60)
```

### 签名易错点
- header values 用 `&` 连接，不是 `;`
- header value 必须 URL-encode（`text/markdown` → `text%2Fmarkdown`）
- header keys 必须小写且按字母排序
- `q-url-param-list=` 留空（不是 `q-url-param-list=&`）

## PushPlus 微信推送（防乱码）

**⚠️ 不能用 `curl -d '{"content":"中文..."}'` 内联方式，Git Bash 会破坏中文编码！**

### 推荐方案：Python requests（最稳）
```python
import requests
resp = requests.post("http://www.pushplus.plus/send", json=payload, timeout=10)
```

### 备选方案：curl + JSON 文件

```json
{
  "token": "<YOUR_PUSHPLUS_TOKEN>",
  "title": "🌟 周复盘 07.19-07.25",
  "content": "<markdown 精简版内容>",
  "template": "markdown"
}
```

```bash
curl -s -X POST "http://www.pushplus.plus/send" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @push_payload.json
```

## 风格要求
- **诚实 > 好看**。像一个认真读完你笔记的朋友，不是夸夸群机器人
- 可以直说"这周没什么产出"，可以直说"这个卡点你已经卡了三周了"
- 只引用知识库内真实内容——任何编造都算任务失败
- 微信精简版 500-1000 字，适合手机阅读，含：一句话实况 + 五维度结论（带证据数字）+ 1-2 个卡点 + 2-4 条建议 + "完整报告见知识库 › 周复盘存档"
- 完整版以知识库存档为准

## 已知问题与修复记录
- 2026-07-25: PushPlus 推送中文乱码 → 改用文件方式（`-d @file.json`）解决
- 2026-07-25: COS 签名 SignatureDoesNotMatch → header values 用 `&` 连接 + URL-encode 解决
- 2026-07-26: get_knowledge_list 子文件夹报"不存在" → 改用 search_knowledge 搜索替代
- 2026-07-26: COS 手动签名 403 → 推荐用 cos-python-sdk-v5 替代
- 2026-07-26: PowerShell Invoke-RestMethod 推送 PushPlus 编码异常 → 用 Python requests 替代
- 2026-07-26: 存档文件夹 get_knowledge_list 返回空但实际已入库 → API 缓存延迟
- 2026-08-09: **用户反馈"报告水分大、看不到真实痛点"** → 根因是只用了 `get_knowledge_list` 的 `introduction`（ima 自动摘要），没读原文。改为**强制 `fetch_media_content` 读正文** + 建素材清单 + 结论必须可追溯，并加入反水分禁令
- 2026-08-09: `get_knowledge_list` 的 `limit` 是 **required** 参数，旧调用示例漏写 → 补全，并改用 `sort_type="UPDATE_TS_DESC_SORT_TYPE"` 倒序取，避免全量遍历
- 2026-08-09: 防重复只查知识库会被缓存延迟坑 → 改为本地文件 + 知识库双校验，存档时先写本地
- 2026-08-09: get_knowledge_list 的 limit 上限是 50（写成 100 报 `value must be inside range (0, 50]`）→ 已把调用示例改为 limit=50
- 2026-08-09: 日记按标题日期归属窗口（如「8.8 利用旧电脑」可能 8.9 晚才入库）→ 不能按 create_time 一刀切，避免漏掉晚入库的窗口内容
- 2026-08-17: 补跑时窗口机械套 [运行日-7, 运行日-1] 会漏掉上周日 → 按「上次报告截止日+1 ~ 最近完整周六」取窗口，与上一份衔接
- 2026-08-17: 历史原件缺某几天的 08-XX 文件夹时，用 `search_knowledge` 搜日期词（如 "8.12"）和主题词确认是真没写还是没归档 → 确认断更后如实写入报告（"8.11~8.16 连续六天零记录"），不推测原因
