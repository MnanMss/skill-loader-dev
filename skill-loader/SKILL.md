---
name: skill-loader
description: >
  项目级 skill 加载器。常驻全局，负责识别“加载”与“维护”两类意图。
  对加载意图：读取配置与本地索引，命中则安装到当前项目；未命中则回退远程搜索。
  对维护意图：不直接执行维护逻辑，而是从 registry 定位对应维护 skill，按需安装后再转交。
  关键词：加载 skill、安装 skill、需要XX功能、有没有 skill 能...
trigger:
  - 用户明确要求"安装到当前项目"、"加载为项目级 skill"、"从 skill 仓库安装"
  - 用户明确要求维护 skill 仓库，例如新增/移除/更新 skill、更新索引、扫描仓库
  - 用户问"项目里有没有对应 skill"、"给当前项目装什么 skill"
---

# Skill Loader

项目级 skill 的常驻加载器。

它的职责只有两件事：

1. 识别用户当前是要“加载 skill”还是“维护 skill 仓库/维护技能集”
2. 在明确意图后，完成项目级安装，或把请求路由到对应维护 skill

## 职责边界

### 负责

- 识别加载意图与维护意图
- 读取配置文件
- 读取本地 skill 仓库索引
- 匹配本地可用 skill
- 将命中的 skill 安装到当前项目
- 根据意图把请求路由到对应维护 skill
- 本地未命中时执行远程搜索 fallback

### 不负责

- 亲自执行重维护逻辑
- 扫描仓库
- 重建 `index.json`
- 新增 skill 到 registry
- 从 registry 减少或移除 skill
- 更新已收录的 skill
- 分类整理 skill
- 修复仓库元数据
- 批量同步 skill
- 修改配置内容

这些维护工作统一交给按需加载的 `skill-registry-maintainer`。

## 配置

skill-loader 优先读取用户配置：

```text
~/.config/skill-loader/config.json
```

Windows：

```text
C:\Users\<username>\.config\skill-loader\config.json
```

### 配置文件结构

```json
{
  "version": "1.0.0",
  "registry": {
    "path": "E:/skills-registry",
    "indexFile": "index.json"
  },
  "behavior": {
    "installScope": "project",
    "autoSearchRemote": true,
    "autoSearchWeb": false,
    "confirmBeforeInstall": false
  },
  "fallback": {
    "remoteRegistry": "skills.sh",
    "searchCommand": "npx skills find"
  },
  "aliases": {
    "word": "docx",
    "excel": "xlsx",
    "ppt": "pptx"
  }
}
```

### 配置项说明

| 配置项 | 说明 |
|--------|------|
| `registry.path` | 本地 skill 仓库根目录 |
| `registry.indexFile` | 索引文件名 |
| `behavior.installScope` | 安装范围，默认 `project` |
| `behavior.autoSearchRemote` | 本地未命中时是否自动搜索远程 |
| `behavior.autoSearchWeb` | 远程未命中后是否继续网页搜索 |
| `behavior.confirmBeforeInstall` | 安装前是否先征求确认 |
| `fallback.remoteRegistry` | 远程 registry 标识 |
| `fallback.searchCommand` | 远程搜索命令 |
| `aliases` | 功能关键词到 skill 名称的映射 |

### 内置默认值

当以上 3 级用户配置均未命中时，skill-loader 使用以下内置默认值：

```json
{
  "version": "1.0.0",
  "registry": {
    "path": "~/skills-registry",
    "indexFile": "index.json"
  },
  "behavior": {
    "installScope": "project",
    "autoSearchRemote": true,
    "autoSearchWeb": false,
    "confirmBeforeInstall": true
  },
  "fallback": {
    "remoteRegistry": "skills.sh",
    "searchCommand": "npx skills find"
  },
  "aliases": {}
}
```

说明：
- `registry.path` 中的 `~` 根据操作系统展开为实际用户目录
- `confirmBeforeInstall: true` 为保守策略，首次使用避免误装
- `aliases` 为空，避免内置别名与全局 skill 冲突

### 配置加载优先级

1. `~/.config/skill-loader/config.json`
2. `./skill-loader/config.json`（项目本地实际配置，建议由 `config.example.json` 复制生成，不提交）
3. `./skill-loader/config.example.json`（示例配置模板）
4. 内置默认值（见上文）

## 意图识别

### 加载意图

只有当用户明确表达“项目级安装”时，才视为加载意图。

示例：

- “帮我把 docx 安装到当前项目”
- “从 skill 仓库给这个项目装一个前端测试 skill”
- “加载为项目级 skill”

以下说法不应默认视为项目级加载意图，而应优先让全局已有 skill 正常匹配：

- “帮我加载 docx skill”
- “我要处理 Word 文档”
- “有没有做 PPT 的 skill”

### 维护意图

以下情况视为维护意图：

- “更新索引”
- “扫描仓库”
- “重建 index.json”
- “检查 skill 仓库配置”
- “整理一下 registry”
- “新增一个 skill 到仓库”
- “把某个 skill 从仓库移除”
- “更新某个已收录 skill”

如果识别为维护意图，不要自己执行维护逻辑，直接转交：

> 这是 skill 仓库维护操作。  
> `skill-loader` 负责识别并路由，具体维护由按需加载的 `skill-registry-maintainer` 执行。

## 加载流程

### Step 1：读取配置

按以下优先级读取配置：

1. `~/.config/skill-loader/config.json`
2. `./skill-loader/config.json`
3. `./skill-loader/config.example.json`
4. 内置默认值

**首次使用引导：**

如果前 3 级配置均未找到（没有任何用户自定义配置），进入首次使用引导流程：

1. **告知用户**：
   > 首次使用 skill-loader，尚未检测到配置文件。skill-loader 负责识别项目级 skill 安装意图与维护意图，并从本地 registry 加载 skill。

2. **询问 registry 路径**：
   > 你希望将本地 skill 仓库放在哪个目录？
   > 默认建议：
   > - Windows：`C:/Users/<username>/skills-registry`
   > - macOS / Linux：`~/skills-registry`

   等待用户回复。若用户接受默认或给出路径，记录下来。

3. **询问是否持久化配置**：
   > 是否为你创建配置文件 `~/.config/skill-loader/config.json`？
   > （推荐创建，否则每次使用内置默认值）

   - 若用户同意：使用 Write 工具创建配置文件，使用用户指定的 registry 路径 + 内置默认值填充其余字段
   - 若用户拒绝：继续使用内置默认值，本次会话生效

4. **检查 registry 目录**：
   如果用户指定的 registry 路径不存在，询问：
   > registry 目录 `<path>` 不存在，是否自动创建？

   若同意，创建目录。

5. 引导完成后，继续后续加载流程。

**配置文件已存在时：**

直接读取生效配置，获取：

- `registry.path`
- `registry.indexFile`
- `behavior.confirmBeforeInstall`
- `behavior.autoSearchRemote`
- `aliases`

### Step 2：解析需求

从用户输入中提取目标 skill 名称或功能关键词，并先判断用户属于“明确点名”还是“模糊需求”。

#### 明确点名

以下情况视为明确点名：

- 用户直接说出 skill 名称
- 用户明确要求把某个已知 skill 安装到当前项目
- 用户给出唯一且明确的项目级目标

示例：

- “帮我把 docx 安装到当前项目”
- “安装 webapp-testing 到当前项目”
- “给当前项目装 mcp-builder”

#### 模糊需求

以下情况视为模糊需求：

- 用户明确要求当前项目需要某种能力，但没有指定具体 skill 名称
- 用户只说想给当前项目加什么能力，没有指定具体 skill
- 用户说法可能命中多个项目级 skill

示例：

- “给当前项目装一个前端测试 skill”
- “从 skill 仓库给这个项目找个文档协作 skill”
- “给这个项目加一个做表格处理的 skill”

优先使用 `aliases` 做一次归一化。

### Step 3：读取本地索引

索引路径：

```text
<registry.path>/<registry.indexFile>
```

如果索引不存在：

> 本地 skill 仓库索引不存在。  
> 请使用 `skill-registry-maintainer` 扫描仓库并重建索引。

### Step 4：匹配本地 skill

匹配优先级：

1. `name` 精确匹配
2. `aliases` 映射后匹配
3. `keywords` 包含匹配
4. `description` 模糊匹配

结果处理要结合用户意图类型：

| 情况 | 行为 |
|------|------|
| 明确点名，且命中 1 个 | 直接进入安装，不再确认 |
| 明确点名，但命中多个 | 列出候选，让用户选择 |
| 模糊需求，命中 1 个或多个 | 列出候选，让用户确认后再安装 |
| 未命中 | 进入远程 fallback |

### Step 5：向用户确认候选（仅模糊需求时）

如果用户最初表达的是能力需求，而不是明确点名某个 skill，则即使本地已经找到候选，也不要直接安装。

应先列出匹配到的 skill，并让用户确认。

示例：

> 找到以下可能适合的 skill：  
> 1. `docx` — Word 文档创建与编辑  
> 2. `pdf` — PDF 文档创建与编辑  
> 请确认要安装哪一个。

如果用户随后明确指定其中一个，再执行安装。

## 项目级安装

安装命令：

```bash
npx skills add <registry.path>/<skill.path> -y
```

规则：

- 默认项目级安装，不加 `-g`
- 只有用户明确要求“安装到当前项目”或“从 skill 仓库加载”时，才执行项目级安装
- 如果用户是明确点名某个 skill，且本地只命中 1 个，则直接安装
- 如果用户是模糊需求，则必须先列出候选并等待确认
- 普通能力请求或普通 skill 名称请求，不应由 skill-loader 抢占，应优先让全局已有 skill 正常匹配
- `behavior.confirmBeforeInstall` 只用于额外控制安装前确认；但它不应覆盖“明确点名直接安装”的主规则
- 安装成功后明确告知已装到当前项目

## 远程 fallback

仅在以下条件满足时触发：

- 本地索引未命中
- `behavior.autoSearchRemote=true`

搜索命令：

```bash
npx skills find <关键词> -y
```

如果远程找到候选：

> 本地仓库没有找到匹配 skill，但远程 registry 中存在候选。  
> 可以继续安装远程 skill，或先收录到本地仓库再安装。

如果远程也没有命中：

> 本地仓库和远程 registry 都没有找到匹配 skill。  
> 可以尝试更宽泛的关键词，或手动创建新 skill。

## 设计原则

- 全局常驻的只有“加载能力”
- “维护能力”必须按需加载，避免日常上下文膨胀
- 普通 skill 请求优先让全局 skill 匹配，项目级安装必须由用户显式提出
- 本地仓库优先，远程搜索兜底
- 明确点名时直接安装，模糊需求时先确认候选
- 默认项目级安装，不污染全局
- 配置驱动，不硬编码用户环境
