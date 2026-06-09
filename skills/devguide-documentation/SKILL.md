---
name: devguide-documentation
description: >-
  Creates and maintains xituan_agent devGuide documentation in Chinese—chooses
  folder placement, deduplicates against existing docs, updates cross-links, avoids
  large code dumps. Use when the user asks to write devGuide, design docs, runbooks,
  implementation guides, or when moving/merging documentation under xituan_agent/devGuide.
---

# devGuide 文档编写

与用户对话仍用中文。本 skill 管 **`xituan_agent/devGuide/`** 定稿文档；未定稿调研放 **`xituan_agent/docs/`**（见 `agent-global-dev-rules-checklist.md` §1）。

## 何时用 devGuide vs docs

| 类型 | 位置 |
|------|------|
| 已定稿方案、实施计划、运维 runbook、长期架构 | `xituan_agent/devGuide/` |
| 讨论中、未收敛、临时调查 | `xituan_agent/docs/` |
| 跨项目契约表、规范 | `xituan_agent/devStandard/` |

**未经用户明确要求，不要新建 `.md`。**

---

## 写作规范

1. **默认中文**（标题、正文、表格说明）；保留英文仅限：API 路径、环境变量、命令、文件名、AWS/产品专有名词。
2. **少用大段纯代码**：
   - 优先：表格、列表、流程说明、架构关系（可用 mermaid）。
   - 代码仅作**短示例**（单行 env、key 命名、目录树 `text` 块 ≤15 行）。
   - 完整实现指到源码路径或 ticket，不要复制整文件。
3. 文首：`Last updated: YYYY-MM-DD`。
4. 文末或文首：**相关文档**链接（相对路径），避免孤岛文档。

---

## 新建 / 扩充文档工作流（必做）

### Step 1 — 选存放位置（先查文件夹，不要直接堆根目录）

1. 列出 `xituan_agent/devGuide/` 及 **一级、二级子目录**（如 `redis_search/redis/`、`redis_search/search/`）。
2. 判断主题是否属于已有 **主题目录**：
   - 已有子项目 README（如 `redis_search/README.md`）→ 优先放进对应子目录，并更新 README 索引表。
   - 已有 **系列约定**（如 `TplPage-*`）→ 遵循该系列命名，见 `TplPage-Documentation-Convention.md`。
   - 与 `aws-setup/` 强相关运维 → 可链到 `xituan_agent/aws-setup/*.md`，devGuide 写「为什么/应用层」，CloudFormation 细节放 aws-setup。
3. 仅当无合适子目录且主题将长期扩展时，才在 `devGuide/` 根下新建文件或新建子文件夹（含 `README.md` 索引）。

**命名**：语义清晰；系列文档用统一前缀；子目录内 as-is 现状可用 `current-state.md`，to-be 方案可用 `*-development.md` / `*-design.md` / `runbook.md`。

### Step 2 — 扫重复（必做，再动笔）

1. 用主题关键词在 `xituan_agent/devGuide/` **搜索**（grep / 文件名 glob）。
2. 打开 **最相关的 1–3 篇** 快速浏览：是否已有相同结论、表格、路线图。
3. 决策（写入 PR/回复中一句话说明）：

| 情况 | 做法 |
|------|------|
| 老文档已完整覆盖 | **不新建**；仅补充老文档缺失的一小段 + 更新 Last updated |
| 老文档是总览，新内容是子专题 | **新文档放子目录**；老文档改为一节摘要 + **链接**到新文档，删掉重复长段 |
| 新内容是总览，老文档是细节 | **新 README/总览**；老文档保留，总览只写结论表 + 链接 |
| 两文重复 >30% | **合并**到更合适的一篇，另一篇改为短 redirect 段或删除（删除前改所有 inbound 链接） |

**禁止** 同一事实在两篇 devGuide 里各写一份详细版；一篇 **canonical（权威）**，其他 **链接引用**。

### Step 3 — 写 / 改内容

- 结构建议：背景一句 → 结论表 → 分节细节 → 相关文档 → （可选）实现 ticket 顺序。
- 与 **Skill** 配套时：devGuide 写全量；`.cursor/skills/*/SKILL.md` 只保留 checklist + 链到 devGuide（见 `redis-valkey-backend` 模式）。

### Step 4 — 更新引用（与 Step 2 决策一起完成）

至少检查并更新：

- 同主题 **README / 总览** 的文档结构表
- **`agent-global-dev-rules-checklist.md`**（仅当新增全仓级文档类别时）
- 相关 **Skill** 的 Related docs 路径
- **`backend-protection-layers-*`**、**`aws-setup/*`** 等交叉主题文档的「相关文档」节
- 若从 A 迁到 B：**全文搜索旧路径**，全部改为新路径

---

## 子目录维护约定（沿用 redis_search 范例）

| 文件 | 用途 |
|------|------|
| `README.md` | 主题总览、架构图、文档索引、落地顺序 |
| `*/current-state.md` | **as-is** 代码/infra 现状 |
| `*-development.md` / `*-design.md` | **to-be** 规划、规范、迁移 |

父 README **只增链接**，不复制子文档长文。

---

## 与其他 skill 的关系

| Skill | 关系 |
|-------|------|
| `product-metadata-schema` | metadata / 搜索：devGuide 细节在 `devGuide/redis_search/` 与 metadata 计划文档 |
| `redis-valkey-backend` | Redis 实现规范 → `devGuide/redis_search/redis/valkey-development.md` |
| `xituan-codebase-change-scope` | 改契约时 devGuide 与 codebase 同步，但不替代 multirepo sync |

---

## Agent checklist（创建 devGuide 时逐项打勾）

- [ ] 已确认用户需要 **定稿 devGuide**（不是 docs 调研）
- [ ] 已扫描 `devGuide/` 及子目录，选定 **canonical 位置**
- [ ] 已读相关旧文，决定 **新写 / 扩充 / 合并 / 仅加链接**
- [ ] 正文 **中文**；无大段纯代码
- [ ] 已更新 README / 总览 / 交叉引用 / Skill 路径
- [ ] 若迁移文件：旧路径无残留引用（repo 内搜索旧文件名）
