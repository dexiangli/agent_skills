# Research Squad — 科研学术团（多角色专家提示词包）

覆盖**化学 / 材料科研全流程**的多角色专家团队：从选题调研到论文投稿。一个主理人统筹，四个专业成员分工，每个成员是独立的角色定义文件——**不绑定任何特定 AI 平台**。

## 团队成员

| 文件 | 角色 | 名字 | 职责 |
|---|---|---|---|
| `agents/research-squad-team-lead.md` | 科研总监（主理人） | 顾全之 | 拆解任务、调度成员、汇总成果、质量把关 |
| `agents/literature-researcher.md` | 文献研究员 | 文思远 | 检索、综述、研究空白分析、背景速通卡 |
| `agents/experiment-advisor.md` | 实验方案顾问 | 方得准 | 方案设计、条件梯度、可行性评估 |
| `agents/data-analyst.md` | 数据分析师 | 数理通 | 清洗、基线校正、峰提取、统计出图 |
| `agents/paper-writer.md` | 论文写作专家 | 章成文 | 结构、写作、润色、选刊与投稿信 |

## 设计内核

- **专业产出由对应成员完成，主理人只做编排与汇编**——避免一个模型又查文献又写论文又管数据的角色混乱
- 每个成员有**标准工作流程（SOP）**和**输出规范**，跨会话行为一致
- 主理人文件内含完整编排逻辑（Workflow A 选题调研 / B 实验方案 / C 数据处理 / D 论文写作），可当"团队协作提示词"范本读

## 怎么用（跨平台适配）

### WorkBuddy（原生格式）
本目录即专家包源格式（`plugin.json` + `agents/`），用专家管理功能导入即可。

### Claude Code / Cursor 等 subagent 体系
把 `agents/*.md` 拷到对应 agents 目录，frontmatter 按目标平台微调：

```markdown
---
name: literature-researcher        # 保留
description: <保留原 description>   # 保留
# WorkBuddy 特有字段删掉：displayName / profession / maxTurns
# 按平台补：tools（如 web_search）、model 等
---
```

正文（角色设定 + SOP + 输出规范）**原样可用**，不需要改。

### 普通聊天式 AI（无 subagent 体系）
直接把某份文件的正文（`#` 以下部分）作为系统提示词粘贴，即可获得对应专家角色。想多角色协作就按主理人文件的 SOP 流程，在对话里逐个调用成员定义。

### 其他领域
角色名和例子是化学科研，但框架通用——替换职责描述和 SOP，就能变成任何领域的专家团（编程、法律、投资……）。

## 文件清单

- `agents/*.md` — 5 份角色定义（核心资产，MIT）
- `plugin.json` — WorkBuddy 专家壳元数据（仅 WorkBuddy 导入用）
- 头像图未包含（体积大且可由各平台自行生成）
