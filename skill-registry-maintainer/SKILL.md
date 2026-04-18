---
name: skill-registry-maintainer
description: >
  本地 skill 仓库维护器。它是由 skill-loader 路由过来的维护 skill，按需加载，
  用于扫描仓库、重建 index.json、检查配置、发现重复和缺失元数据，以及处理新增、减少、更新 skill。
  仅在用户明确提出维护 skill 仓库时使用。
trigger:
  - 用户说"更新索引"
  - 用户说"扫描仓库"
  - 用户说"重建 index.json"
  - 用户说"检查 skill-loader 配置"
  - 用户说"整理/维护 skill 仓库"
  - 用户说"初始化仓库"
  - 用户说"新增一个 skill 到仓库"
  - 用户说"把某个 skill 从仓库移除"
  - 用户说"更新某个已收录 skill"
---

# Skill Registry Maintainer

本 skill 只负责维护本地 skill 仓库，不负责项目级安装。

它不是常驻 skill，而是按需加载的维护工具。

## 职责范围

### 负责

- 读取并校验 `skill-loader` 配置
- 扫描本地 skill 仓库目录
- 重建 `index.json`
- 新增 skill 到 registry
- 从 registry 减少或移除 skill
- 更新已收录 skill
- 检查重复条目
- 检查缺失的 `name` 或 `description`
- 检查索引路径和目录结构是否一致

### 不负责

- 响应普通加载请求
- 为当前项目安装 skill
- 代替 `skill-loader` 做日常路由

## 共享配置

沿用 `skill-loader` 的配置文件：

```text
~/.config/skill-loader/config.json
```

项目内建议保留两份文件：

- `config.example.json`：提交到仓库，作为模板
- `config.json`：本地实际配置，不提交到仓库

关键配置项：

- `registry.path`
- `registry.indexFile`
- `aliases`

如果配置不存在，`skill-loader` 会提供首次使用引导并给出内置默认值。维护器应能处理由引导生成的配置。

## 仓库约定

默认扫描以下结构：

```text
<registry.path>/
├── official/
├── community/
└── custom/
```

每个 skill 目录下应至少包含：

```text
SKILL.md
```

## 维护流程

### Step 1：读取配置

读取生效配置，确认：

- 仓库目录存在
- 索引输出路径可写
- aliases 可解析

如果配置无效：

> 检测到 skill-loader 配置异常。  
> 请先修正 `registry.path` 或 `indexFile`。

### Step 2：初始化仓库

**触发条件：**

- `registry.path` 目录不存在，或
- `registry.path` 存在但缺少 `official/`、`community/`、`custom/` 中的至少一个

**初始化引导：**

1. **告知用户**：
   > 本地 skill 仓库尚未初始化，需要创建目录结构。

2. **列出将要创建的目录**：
   ```text
   <registry.path>/
   ├── official/
   ├── community/
   └── custom/
   ```

3. **询问是否创建**：等待用户确认

4. **创建目录**：若同意，创建上述目录 + 空 `index.json`

5. **询问是否扫描全局 skill**：
   > 是否扫描你当前的全局 skill，并给出迁移建议？
   > （全局 skill 位于 `~/.claude/skills/`）

   - 若同意，进入全局 skill 扫描与迁移推荐流程
   - 若拒绝，初始化完成，后续可手动触发

### Step 3：全局 skill 扫描与迁移推荐

**扫描范围：**

- `~/.claude/skills/` — 当前生效的全局 skill
- `~/.claude/skills-disabled/` — 已被禁用/移出的全局 skill

**提取信息：**

对每个 skill，读取其 `SKILL.md` 的 frontmatter：
- `name`
- `description`（前 100 字符）
- `source`（如果有）
- 当前状态（active / disabled）

**推荐分类规则：**

| 类别 | 特征 | 示例 | 建议 |
|------|------|------|------|
| A. 官方高频通用 | 来自 anthropic/skills 官方，与文件格式强绑定，跨项目高频使用 | docx, pdf, pptx, xlsx | **保留全局** |
| B. 工具基础设施 | skill-loader 本身、agent-reach 等通用工具 | skill-loader, agent-reach | **保留全局** |
| C. 项目专属 | 与特定业务、项目、技术栈强绑定，低频跨项目使用 | channel-change-oracle, springboot-password-encryption, nesma-analyzer | **移入 registry (custom/)** |
| D. 通用指南类 | 编码准则、代码风格，跨项目可用但不频繁 | karpathy-guidelines | **移入 registry (community/)** 或保留全局 |
| E. 已禁用 | 已在 `skills-disabled/` 中 | summary-skills | **询问是否移入 registry 或删除** |

**决策逻辑（按优先级判断）：**

1. 如果在 `skills-disabled/` 中 → 标记为"已禁用"，建议移入 registry 或清理
2. 如果 `name` 为 docx / pdf / pptx / xlsx → 标记为"官方高频"，建议保留全局
3. 如果 `name` 包含 skill-loader / agent-reach → 标记为"基础设施"，建议保留全局
4. 如果 `description` 中包含特定业务关键词（Oracle, Spring Boot, NESMA 等）→ 标记为"项目专属"，建议移入 `custom/`
5. 如果 `description` 是通用编码/写作指南 → 标记为"通用指南"，建议移入 `community/` 或保留全局
6. 其余 → 标记为"待评估"，由用户决定

**报告格式：**

```text
扫描到 X 个全局 skill，推荐如下：

【建议保留全局】（Y 个）
1. docx — Word 文档处理（官方高频通用）
2. pdf — PDF 文档处理（官方高频通用）
...

【建议移入 registry】（Z 个）
1. channel-change-oracle — Oracle AIADMIN 渠道变更（项目专属）→ 建议放入 custom/
2. karpathy-guidelines — 编码行为准则（通用指南）→ 建议放入 community/
...

【已禁用】（W 个）
1. summary-skills — 摘要总结（已在 skills-disabled/）→ 建议移入 custom/ 或直接删除
...
```

**执行确认：**

列出所有推荐后，询问用户是否按推荐执行迁移。

- 若同意：将建议移入 registry 的 skill **复制**到对应目录（`official/` / `community/` / `custom/`）
- 若拒绝或部分拒绝：尊重用户选择

**安全约定：**

- 迁移是**复制**而非移动，保留原全局 skill 作为备份，由用户手动删除
- 如果目标路径已存在同名 skill，询问是否覆盖
- 迁移完成后，建议用户运行"更新索引"

### Step 4：扫描仓库

递归扫描以下目录中的 `SKILL.md`：

- `official/`
- `community/`
- `custom/`

忽略无 `SKILL.md` 的目录。

### Step 5：提取元数据

优先从 frontmatter 提取：

- `name`
- `description`

生成索引条目时补充：

- `path`：相对 `registry.path` 的路径
- `source`：根据一级目录推断，如 `official`、`community`、`custom`
- `keywords`：按以下顺序生成
  1. frontmatter 中的 `keywords`（如果有）
  2. skill 的 `name`
  3. 命中的 aliases 反向补充词

## 索引输出格式

输出到：

```text
<registry.path>/<indexFile>
```

推荐结构：

```json
{
  "version": "1.0.0",
  "updatedAt": "YYYY-MM-DD",
  "skills": []
}
```

每个条目至少包含：

```json
{
  "name": "docx",
  "source": "official",
  "path": "official/anthropics/docx",
  "keywords": ["docx"],
  "description": "Word 文档创建与编辑"
}
```

## 常见维护指令

### “初始化仓库”

行为：

1. 检查 `registry.path` 是否存在
2. 若不存在或结构不完整，引导用户创建 `official/`、`community/`、`custom/` 目录
3. 询问是否扫描全局 skill 并给出迁移建议
4. 根据用户确认执行迁移（复制到对应目录）
5. 建议运行"更新索引"

### “更新索引”

行为：

1. 扫描仓库
2. 重建 `index.json`
3. 报告新增、删除、重复、缺失项

### “扫描仓库”

行为：

- 只检查目录与 `SKILL.md` 是否齐全
- 不强制改写索引，除非用户明确要求

### “检查配置”

行为：

- 展示当前生效配置
- 检查路径是否存在
- 检查索引文件是否可读写

### “检查索引质量”

行为：

- 查重
- 查缺失 description
- 查 path 与目录不一致的条目

## 维护原则

- 维护与加载解耦
- 扫描结果优先基于事实，不猜测缺失字段
- 仅在用户明确要求时改写 `index.json`
- 尽量保留已有人工维护的关键词，不擅自覆盖
- 报告问题时给出可执行修复建议
