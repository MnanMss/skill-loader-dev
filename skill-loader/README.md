# Skill Loader 配置指南

Skill Loader 是一个常驻全局的轻量入口 skill，负责识别"项目级安装"与"维护"两类意图，并从本地 registry 加载 skill。

本文档面向用户，说明如何配置 skill-loader。

---

## 快速开始

1. 复制模板文件为实际配置：

   ```bash
   cp skill-loader/config.example.json skill-loader/config.json
   ```

2. 修改 `skill-loader/config.json` 中的 `registry.path` 为你本地的 skill 仓库目录。

3. 首次使用时，skill-loader 会检测配置并主动引导你完成设置。你也可以手动创建配置文件：

   - **Windows**：`C:\Users\<username>\.config\skill-loader\config.json`
   - **macOS / Linux**：`~/.config/skill-loader/config.json`

---

## 配置文件位置

skill-loader 按以下优先级读取配置，高优先级覆盖低优先级：

| 优先级 | 路径 | 说明 |
|--------|------|------|
| 1 | `~/.config/skill-loader/config.json` | 用户级配置，推荐长期使用 |
| 2 | `./skill-loader/config.json` | 项目级配置，当前仓库专属 |
| 3 | `./skill-loader/config.example.json` | 示例模板，提交到仓库 |
| 4 | 内置默认值 | 无配置时的兜底，见下方说明 |

> **约定**：`config.example.json` 提交到仓库，`config.json` 不提交。

---

## 配置项详解

### `registry`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | string | 是 | 本地 skill 仓库根目录。例如 `E:/skills-registry` 或 `~/skills-registry` |
| `indexFile` | string | 是 | 索引文件名，通常保持 `index.json` |

### `behavior`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `installScope` | string | `project` | 安装范围。当前仅支持 `project`，即安装到当前项目 |
| `autoSearchRemote` | boolean | `true` | 本地 registry 未命中时，是否自动搜索远程 registry |
| `autoSearchWeb` | boolean | `false` | 远程也未命中时，是否继续网页搜索 |
| `confirmBeforeInstall` | boolean | `true` | 安装前是否征求用户确认。首次使用建议保持 `true` |

### `fallback`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `remoteRegistry` | string | `skills.sh` | 远程 registry 标识 |
| `searchCommand` | string | `npx skills find` | 远程搜索命令 |

### `aliases`

功能关键词到 skill 名称的映射。用于把用户的口语化表达归一化为标准 skill 名。

示例：

```json
{
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

---

## 完整示例

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
    "confirmBeforeInstall": true
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

---

## 最小可用配置

如果你只关心 registry 路径，最小配置如下：

```json
{
  "registry": {
    "path": "~/skills-registry"
  }
}
```

其余字段均使用内置默认值。

---

## 内置默认值

当没有任何用户配置时，skill-loader 使用以下默认值：

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

- `~` 会根据操作系统自动展开
- `confirmBeforeInstall: true` 为保守策略，避免误装

---

## 常见问题

### Q: Windows 下路径怎么写？

推荐写法：

- `E:/skills-registry`（正斜杠，跨平台兼容）
- 避免使用反斜杠 `\`，如需使用请转义为 `\\`

### Q: registry 目录不存在怎么办？

首次使用时，skill-loader 会检测目录是否存在并询问是否自动创建。你也可以手动创建：

```bash
mkdir -p ~/skills-registry/{official,community,custom}
```

### Q: 项目级配置和用户级配置冲突时以谁为准？

以 **高优先级** 为准。`~/.config/skill-loader/config.json`（用户级）优先级最高，会覆盖项目级配置。

### Q: 为什么 `config.json` 不提交到仓库？

因为 `registry.path` 通常包含个人本地路径（如 `E:/skills-registry`），不同用户路径不同。提交模板（`config.example.json`）即可。
