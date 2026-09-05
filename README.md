# My WorkBuddy Skills

个人用的 AI Agent 技能集合，MIT 开源。内容是真实使用里长出来的——每一条规则背后基本都对应一次踩坑记录，不是整理出来的通用方法论。

## 技能列表

### 1. `ima-weekly-review` — 知识库周复盘自动化

从 [ima 知识库](https://ima.qq.com) 扫描上周笔记，生成周复盘报告，存档回知识库，并通过 PushPlus 推送微信精简版。

**解决的真问题**：第一版复盘报告被反馈"水分太大、看不到真实痛点"。根因是 AI 用了 API 返回的摘要字段当素材——摘要已经是自动加工过的一层抽象，在上面再加工一次，产出全是"正确的废话"。

现在的版本强制读笔记正文、要求每个结论可追溯到原句、并写死了一串反水分禁令。另外还有：

- 本地文件 + 云端查询**双校验**防重复生成/重复推送
- 补跑场景的日期窗口衔接算法（避免漏掉或重复某几天）
- 腾讯云 COS 上传的签名实现（含易错点清单）
- Git Bash 下中文 JSON 乱码的解法

→ [`skills/ima-weekly-review/SKILL.md`](skills/ima-weekly-review/SKILL.md)

### 2. `personal-knowledge-base` — 个人知识库搭建与内化

搭一个能真正调用的（而不是只会收藏的）知识库。

核心信条：**你看过的世界不是你，你理解的世界才是你。**

包含四块：

| 模块 | 内容 |
|---|---|
| 什么值得存 | 判断标准（三个条件缺一不可）、存/不存清单、六条核心习惯 |
| 目录结构 | 两套可直接抄的方案：通用成长版 / 研究生科研版，含标签与命名规范 |
| 内化方法 | 三阶段学习法、30 分钟迭代卡片法、知识内化五步 |
| AI 协作防错 | 6 个真实事故 → 硬约束，附自检清单 |

其中「AI 协作防错」这一节比较特别：每一条都对应一次真实事故（AI 没读截图就编笔记、批量移动丢 9 条数据、同一件事被拆成多条笔记等），不是纸上谈兵。

→ [`skills/personal-knowledge-base/SKILL.md`](skills/personal-knowledge-base/SKILL.md)

## 安装

把对应目录复制到你的 skills 目录即可：

```bash
# WorkBuddy
cp -r skills/* ~/.workbuddy/skills/

# Claude Code
cp -r skills/* ~/.claude/skills/
```

用 `ima-weekly-review` 前，需要把 `SKILL.md` 里的占位符替换成你自己的配置：

| 占位符 | 怎么拿 |
|---|---|
| `<YOUR_KB_ID>` | `get_knowledge_base_list` 返回的 `id` |
| `<FOLDER_ID_*>` | `get_knowledge_list` 逐级返回的 `folder_info.folder_id` |
| `<YOUR_PUSHPLUS_TOKEN>` | [pushplus.plus](https://www.pushplus.plus) 登录后免费获取 |

**不要把真实 token 提交进公开仓库。**

## 关于隐私

本仓库所有内容发布前都做过脱敏：密钥、知识库 ID、文件夹 ID、个人标识全部替换为占位符。如果你 fork 后要提交自己的配置，记得改回去。

## License

MIT — 随便用，改了不用告诉我。
