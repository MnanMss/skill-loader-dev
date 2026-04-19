# skill-loader-dev

中文 | [English](#english)

一个用于管理 Claude Code skills 的实验性仓库，核心目标是把 **全局 skill**、**项目级 skill**、**skill 仓库维护** 三件事拆开处理。

这个仓库当前包含两个核心部分：

- `skill-loader`：常驻全局的轻量入口，用于识别**项目级安装意图**和**仓库维护意图**
- `skill-registry-maintainer`：按需安装的维护 skill，用于处理 registry 扫描、索引更新、skill 增删改等操作

## 为什么要做这个

当 skill 变多以后，全部放进全局层会带来几个问题：

- 全局匹配范围越来越大，命中变得不稳定
- 项目专属 skill 会污染其他项目
- 日常使用和仓库维护耦合在一起
- registry 的维护逻辑会让常驻 skill 变得越来越重

这个仓库的思路是：

- **全局层只保留少量高频 skill 和一个入口 skill**
- **大部分 skill 进入本地 registry**
- **只有在用户明确要求时，才把 skill 安装到当前项目**
- **维护逻辑完全拆到独立 maintainer skill**

## 架构概览

```text
用户请求
   │
   ▼
全局 skill 匹配
   │
   ├── 命中普通全局 skill（如 docx / pdf / xlsx）→ 直接使用
   │
   └── 命中 skill-loader
         │
         ├── 项目级安装意图 → 查本地 registry → 候选确认/直接安装 → 安装到当前项目
         │
         └── 仓库维护意图 → 路由到 skill-registry-maintainer → 按需执行维护
```

## 设计原则

- **全局优先**：普通 skill 请求应优先让现有全局 skill 自然匹配
- **项目级显式触发**：只有用户明确说“安装到当前项目”“从 skill 仓库安装”时，才进入项目级安装流程
- **维护与加载解耦**：`skill-loader` 负责识别和路由，不直接做重维护
- **本地优先，远程兜底**：本地 registry 命中优先，找不到时再用远程搜索
- **模板可提交，实际配置不提交**：提交 `config.example.json`，本地使用 `config.json`

## 仓库结构

`skill-loader-dev` 是**开发仓库**，只保存 `skill-loader` 与 `skill-registry-maintainer` 的定义、文档和示例配置；它不承载实际运行时的 skill 数据。

```text
skill-loader-dev/
├── README.md
├── .gitignore
├── skill-loader/
│   ├── SKILL.md
│   ├── README.md
│   └── config.example.json
└── skill-registry-maintainer/
    ├── SKILL.md
    └── config.example.json
```

配套的本地 registry 应作为**独立的运行时仓库**存在，建议使用单独的 Git 仓库进行版本管理，例如 `E:/skills-registry`：

```text
skills-registry/
├── .git/
├── .gitignore
├── index.json
├── official/
├── community/
└── custom/
```

约定：

- `skill-loader-dev` 与 `skills-registry` 分离维护，不要把运行时 registry 放进开发仓库中
- `index.json` 是 runtime registry 的可消费索引，也是发布产物，应提交到 `skills-registry`
- `skills-registry/.gitignore` 只应忽略编辑器或系统噪音文件，不应忽略 `index.json`、skill 目录或 `SKILL.md`

## 组件说明

### 1. skill-loader

`skill-loader` 是一个**全局入口 skill**，职责很轻：

- 识别用户是否在请求**项目级安装**
- 识别用户是否在请求**维护 registry**
- 读取配置与本地索引
- 在用户明确指定时，把 skill 安装到当前项目
- 把维护意图路由给 `skill-registry-maintainer`

它不负责：

- 扫描 registry
- 重建 `index.json`
- 新增/移除/更新 registry 中的 skill
- 直接执行重维护逻辑

### 2. skill-registry-maintainer

`skill-registry-maintainer` 是一个**按需安装的维护 skill**。

它负责：

- 扫描 registry
- 重建 `index.json`
- 校验配置
- 检查缺失元数据与重复项
- 新增、移除、更新 registry 中的 skill

它不负责：

- 响应普通使用场景
- 代替 `skill-loader` 做日常入口
- 处理普通项目级安装请求

## 工作流

### 工作流 A：安装 skill 到当前项目

当用户明确提出项目级安装需求时：

1. `skill-loader` 识别为项目级安装意图
2. 读取配置和本地 `index.json`
3. 在 registry 中查找 skill
4. 根据请求类型决定是否确认：
   - **明确点名 skill**：若命中唯一 skill，直接安装
   - **能力型需求**：列出候选，等待用户确认
5. 执行项目级安装

安装命令示例：

```bash
npx skills add <registry.path>/<skill.path> -y
```

### 工作流 B：维护 skill 仓库

当用户提出维护请求时：

1. `skill-loader` 识别为维护意图
2. 不自己执行维护逻辑
3. 将请求路由到 `skill-registry-maintainer`
4. 由 maintainer 负责扫描、更新、增删改、索引检查等操作

## 触发边界

### 应该触发 `skill-loader` 的请求

- `把 webapp-testing 安装到当前项目`
- `从 skill 仓库给这个项目找个前端测试 skill`
- `更新 registry 索引`
- `把某个 skill 从仓库里移除`

### 不应该默认触发 `skill-loader` 的请求

- `用 docx`
- `我要处理 Word 文档`
- `有没有做 PPT 的 skill`

这些请求应该优先交给已经存在的全局 skill，如 `docx`、`pdf`、`pptx`、`xlsx`。

## 配置约定

配置优先级：

1. `~/.config/skill-loader/config.json`
2. `./skill-loader/config.json`
3. `./skill-loader/config.example.json`
4. 内置默认值

示例配置：

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
    "ppt": "pptx",
    "powerpoint": "pptx",
    "表格": "xlsx",
    "文档": "docx",
    "幻灯片": "pptx",
    "pdf处理": "pdf"
  }
}
```

### 提交约定

应该提交：

- `config.example.json`

不应该提交：

- `config.json`

推荐流程：

1. 提交模板文件 `config.example.json`
2. 本地复制为 `config.json`
3. 修改本地路径，不提交真实配置

## 使用示例

### 示例 1：明确点名安装到当前项目

用户：

```text
把 webapp-testing 安装到当前项目
```

行为：

- 命中 `skill-loader`
- 识别为明确点名 + 项目级安装
- 如果本地唯一命中，则直接安装

### 示例 2：能力型项目需求

用户：

```text
从 skill 仓库给这个项目找个文档协作 skill
```

行为：

- 命中 `skill-loader`
- 识别为模糊能力需求
- 列出候选 skill
- 等待用户确认后安装

### 示例 3：维护 registry

用户：

```text
更新 registry 索引
```

行为：

- 命中 `skill-loader`
- 识别为维护意图
- 路由到 `skill-registry-maintainer`

## 当前状态

当前方案已经形成以下边界：

- `skill-loader` 作为全局入口长期存在
- 一部分更适合项目专属的 skill 已从全局层移出，保留在 registry 中
- `skill-registry-maintainer` 负责独立的维护逻辑
- 项目级安装必须由用户显式提出，避免抢占全局 skill 的正常匹配

## 后续可扩展方向

- 为 registry 建立自动索引更新脚本
- 增加 registry 质量检查命令
- 支持更丰富的 skill 分类与标签体系
- 增加远程 registry 同步策略
- 为 `skill-loader` 增加更严格的防冲突规则

---

## English

`skill-loader-dev` is an experimental repository for managing Claude Code skills with a split architecture:

- keep a **small global layer**
- move most skills into a **local registry**
- install skills into a project **only when explicitly requested**
- separate **registry maintenance** from the always-on loader

This repository currently contains two core skills:

- `skill-loader`: a lightweight global entrypoint for project-scoped installs and maintenance routing
- `skill-registry-maintainer`: an on-demand maintainer skill for registry operations such as scanning, indexing, and skill add/remove/update

### Why this exists

As the number of skills grows, putting everything in the global layer causes problems:

- global matching becomes noisy and less predictable
- project-specific skills leak into unrelated work
- maintenance workflows get mixed with day-to-day usage
- the global entrypoint becomes too heavy

This repository explores a cleaner pattern:

- keep only a few high-frequency skills global
- store most skills in a local registry
- install a skill into the current project only when the user explicitly asks for it
- move maintenance logic into a separate maintainer skill

### Architecture overview

```text
User request
   │
   ▼
Global skill matching
   │
   ├── Match a normal global skill (docx / pdf / xlsx / etc.) → use it directly
   │
   └── Match skill-loader
         │
         ├── Project install intent → search local registry → confirm or install → install into current project
         │
         └── Maintenance intent → route to skill-registry-maintainer → run maintenance on demand
```

### Core principles

- **Global-first**: normal skill requests should be handled by existing global skills whenever possible
- **Explicit project install**: project-scoped installs should happen only when the user clearly asks for them
- **Maintenance/install separation**: `skill-loader` routes, `skill-registry-maintainer` maintains
- **Local-first, remote fallback**: prefer the local registry before remote search
- **Commit templates, not local config**: commit `config.example.json`, keep `config.json` local

### Repository layout

`skill-loader-dev` is the **development repository**. It only stores the definitions, docs, and example configs for `skill-loader` and `skill-registry-maintainer`; it should not contain runtime skill data.

```text
skill-loader-dev/
├── README.md
├── .gitignore
├── skill-loader/
│   ├── SKILL.md
│   ├── README.md
│   └── config.example.json
└── skill-registry-maintainer/
    ├── SKILL.md
    └── config.example.json
```

The local registry should exist as a **separate runtime repository**, ideally managed as its own Git repository, for example `E:/skills-registry`:

```text
skills-registry/
├── .git/
├── .gitignore
├── index.json
├── official/
├── community/
└── custom/
```

Conventions:

- keep `skill-loader-dev` and `skills-registry` separate — do not place the runtime registry inside the development repo
- `index.json` is the consumable runtime index and should be committed in `skills-registry`
- `skills-registry/.gitignore` should ignore only editor/OS noise, not `index.json`, skill directories, or `SKILL.md`

### Components

#### skill-loader

`skill-loader` is the always-available global entrypoint.

It is responsible for:

- detecting whether the request is a **project install** or a **maintenance action**
- reading config and the local registry index
- installing a matching skill into the current project when appropriate
- routing maintenance requests to `skill-registry-maintainer`

It is not responsible for:

- scanning the registry
- rebuilding `index.json`
- directly adding/removing/updating registry entries
- performing heavy maintenance itself

#### skill-registry-maintainer

`skill-registry-maintainer` is an on-demand maintenance skill.

It is responsible for:

- scanning the registry
- rebuilding `index.json`
- validating configuration
- checking duplicates and missing metadata
- adding, removing, and updating registry skills

It is not responsible for:

- normal end-user task handling
- replacing `skill-loader` as the daily entrypoint
- handling standard project install requests

### Workflow

#### Workflow A: install a skill into the current project

When the user explicitly asks for a project-scoped install:

1. `skill-loader` recognizes the request as project-install intent
2. it reads config and local `index.json`
3. it searches the local registry
4. it decides whether confirmation is needed:
   - **explicit named skill**: install directly if there is exactly one match
   - **capability request**: list candidates and wait for confirmation
5. it installs the selected skill into the current project

Example install command:

```bash
npx skills add <registry.path>/<skill.path> -y
```

#### Workflow B: maintain the skill registry

When the user asks for registry maintenance:

1. `skill-loader` recognizes maintenance intent
2. it does not run the maintenance logic itself
3. it routes the request to `skill-registry-maintainer`
4. the maintainer handles scanning, indexing, add/remove/update, and quality checks

### Trigger boundaries

Requests that should trigger `skill-loader`:

- `Install webapp-testing into the current project`
- `Find a frontend testing skill for this project from the registry`
- `Update the registry index`
- `Remove a skill from the registry`

Requests that should not be treated as project-install requests by default:

- `Use docx`
- `I want to edit a Word document`
- `Is there a skill for PPT?`

These should normally be handled by existing global skills such as `docx`, `pdf`, `pptx`, and `xlsx`.

### Configuration

Configuration resolution order:

1. `~/.config/skill-loader/config.json`
2. `./skill-loader/config.json`
3. `./skill-loader/config.example.json`
4. built-in defaults

Example configuration:

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
    "ppt": "pptx",
    "powerpoint": "pptx",
    "表格": "xlsx",
    "文档": "docx",
    "幻灯片": "pptx",
    "pdf处理": "pdf"
  }
}
```

### Config commit rules

Commit this:

- `config.example.json`

Do not commit this:

- `config.json`

Recommended workflow:

1. keep `config.example.json` in version control
2. copy it locally to `config.json`
3. customize local paths without committing them

### Example interactions

#### Install a named skill into the project

User:

```text
Install webapp-testing into the current project
```

Behavior:

- match `skill-loader`
- classify as explicit named project install
- install directly if there is exactly one local match

#### Ask for a capability

User:

```text
Find a document collaboration skill for this project from the registry
```

Behavior:

- match `skill-loader`
- classify as a capability-based project request
- list candidates
- wait for user confirmation
- install the selected skill

#### Ask for maintenance

User:

```text
Update the registry index
```

Behavior:

- match `skill-loader`
- classify as maintenance intent
- route to `skill-registry-maintainer`

### Current status

The repository currently follows this split model:

- `skill-loader` remains the stable global entrypoint
- some project-specific skills have been removed from the global layer and kept in the registry
- `skill-registry-maintainer` owns the maintenance workflows
- project installs must be explicitly requested so normal global matching is not stolen

### Future directions

- automate registry index generation
- add registry quality-check commands
- support richer tagging and classification
- improve remote registry sync strategies
- add stricter conflict-avoidance rules to `skill-loader`

## License

Add your preferred license here.
