---
name: orcanote-plugin-dev-skills
description: 开发虎鲸笔记（Orca Note）插件的完整工作流。包含三大流程：1) 生成项目——基于 sethyuan/orca-plugin-template 模板创建插件项目骨架并复制 plugin-docs API 文档；2) 测试插件——构建后将必要文件同步到本地 Orca 插件目录并重启 Orca Note；3) 发布版本——在开发分支上按 x.x.x 格式递增版本号、构建发布包、维护 awesome-orcanote 仓库的 plugins.json 插件市场注册表（不操作 main 分支，合并由用户在 GitHub 上发起 PR 完成）。当用户要求创建/初始化虎鲸笔记插件项目，或要求测试、安装插件到本地虎鲸笔记，或要求发布、上架、更新虎鲸笔记插件版本时使用。
---

# 开发虎鲸笔记插件技能

虎鲸笔记（Orca Note）插件的项目生成、本地测试与版本发布工作流。

工作区可能是多 module 模式（根目录下多个独立 git 仓库的插件子项目），也可能本身就是单个插件项目。执行前先判断：若用户指定了子项目则操作该子目录；否则若当前目录含 `package.json` + `src/main.ts` 则视为单项目模式。

---

## 流程一：生成项目

```
Task Progress:
- [ ] Step 1: 获取官方模板
- [ ] Step 2: 创建项目目录并拷贝骨架
- [ ] Step 3: 复制 plugin-docs 文档
- [ ] Step 4: 初始化项目信息
```

**Step 1: 获取官方模板**

优先复用本地缓存，避免重复克隆（GitHub 直连可能超时）：

```bash
# 已有缓存则跳过克隆
ls /tmp/orca-plugin-template 2>/dev/null || \
  git clone --depth 1 git@github.com:sethyuan/orca-plugin-template.git /tmp/orca-plugin-template
# SSH 失败时回退 https://github.com/sethyuan/orca-plugin-template.git
```

**Step 2: 创建项目目录并拷贝骨架**

在工作区根目录下创建新插件目录（名称由用户指定，小写字母+连字符），拷贝模板内容但排除 `.git`：

```bash
mkdir <plugin-name> && \
  rsync -a --exclude .git /tmp/orca-plugin-template/ <plugin-name>/
```

**Step 3: 复制 plugin-docs 文档**

模板中已含 `plugin-docs/`（Orca 插件 API 的离线 Markdown 文档），确认拷贝完整（应有 9 个文件：`modules.md`、`documents/` 5 篇指南、`types/orca.md`、`constants/` 2 个）。若模板中缺失，从工作区其他插件项目复制。

**Step 4: 初始化项目信息**

1. 修改 `package.json` 的 `name`、`description`（`version` 置为 `0.1.0`）
2. 创建 `AGENTS.md`，内容参考工作区其他插件项目的同名文件（指引先读 `plugin-docs` 再编码）
3. `git init` 并创建首次提交，默认分支 `main`，开发分支建议 `dev`
4. 提醒用户：需准备 `icon.png` 或 `icon.svg`（≤80x80 px）和 `LICENSE` 文件，发布上架时必需

---

## 流程二：测试插件

```
Task Progress:
- [ ] Step 1: 构建插件项目
- [ ] Step 2: 同步到本地 Orca 插件目录
- [ ] Step 3: 重启 Orca Note
```

本地 Orca 插件目录：`<Orca 数据目录>/orca/plugins/<plugin-name>/`（目录名与插件项目同名）。macOS 默认为 `~/Documents/orca/plugins/`；实际位置以 Orca Note 设置中的数据目录为准。

**Step 1: 构建插件项目**

```bash
cd <project> && npm run build
```

**Step 2: 同步到本地 Orca 插件目录**

只复制运行必需文件（`dist/`、`package.json`、`icon.png`、`LICENSE`），排除 `node_modules`、`src`、`.git` 等开发文件：

```bash
mkdir -p ~/Documents/orca/plugins/<plugin-name> && \
  rsync -a --delete \
    --include='dist/***' --include='package.json' \
    --include='icon.png' --include='LICENSE' --exclude='*' \
    <project>/ ~/Documents/orca/plugins/<plugin-name>/
```

同步后用 `ls` 核对目标目录内容。

**Step 3: 重启 Orca Note**

macOS：

```bash
osascript -e 'quit app "Orca Note"'; sleep 2; open -a "Orca Note"
```

其他平台手动重启 Orca Note 即可。重启后提醒用户在 Orca Note 中验证插件功能；若插件未生效，检查插件是否已在设置→插件中启用。

---

## 流程三：发布版本

```
Task Progress:
- [ ] Step 1: 确定目标项目与新版本号
- [ ] Step 2: 在开发分支上提交版本号
- [ ] Step 3: 构建并打发布包（zip）
- [ ] Step 4: 准备 awesome-orcanote 仓库
- [ ] Step 5: 更新 plugins.json
```

> 发布全程在**开发分支**（如 `develop`）上进行，**不操作 main 分支**：合并由用户在 GitHub 页面单独发起 PR 完成，tag/Release 由用户在 PR 合并后于 GitHub 上创建（创建 Release 时 GitHub 会自动打 tag）。

**Step 1: 确定目标项目与新版本号**

1. 确定目标项目目录（用户指定的子项目，或当前项目）
2. 查看已有 tag 决定新版本号：

```bash
git -C <project> tag --sort=-v:refname | head -5
```

版本格式 `x.x.x`。默认在最新 tag 基础上递增 patch 位（如 `1.2.3` → `1.2.4`）；若无任何 tag，从 `1.0.0` 开始；用户明确要求时递增 minor/major 位。

**Step 2: 在开发分支上提交版本号**

```bash
git -C <project> status --short          # 确认工作区干净，有未提交改动先询问用户
git -C <project> branch --show-current   # 确认当前在开发分支（如 develop），不切换到 main
```

同步更新项目 `package.json` 的 `version` 字段并提交到开发分支（发布包校验依赖此字段）。

**Step 3: 构建并打发布包（zip）**

zip 是 plugins.json 中 `zip` 字段指向的插件市场安装包，**必须包含**：`package.json`（含 version 字段）、`dist/` 构建产物、`LICENSE`、`icon.png` 或 `icon.svg`。

**zip 内部结构必须是以插件名命名的顶层目录**（不带版本号，如 `my-plugin/`），否则虎鲸笔记解压后无法正确识别插件；zip 文件名本身可带版本号（`<plugin-name>-v<x.x.x>.zip`）：

```bash
cd <project> && npm run build && \
  rm -rf /tmp/<plugin-name>-pack && mkdir -p /tmp/<plugin-name>-pack/<plugin-name> && \
  cp -R dist package.json LICENSE icon.png /tmp/<plugin-name>-pack/<plugin-name>/ && \
  (cd /tmp/<plugin-name>-pack && zip -r <project绝对路径>/<plugin-name>-v<x.x.x>.zip <plugin-name>)
```

打包后用 `unzip -l` 核对：所有条目均应位于 `<plugin-name>/` 之下。提醒用户将 zip 上传到该项目 GitHub Release（tag 为 `v<x.x.x>`），下载链接格式：
`https://github.com/<owner>/<repo>/releases/download/v<x.x.x>/<zip-name>.zip`

**Step 4: 准备 awesome-orcanote 仓库**

在工作区根目录查找 `awesome-orcanote` 目录：

```bash
ls awesome-orcanote 2>/dev/null
```

- 存在：`git -C awesome-orcanote pull` 同步最新（失败不阻塞，继续用本地版本）
- 不存在：fork 并克隆到工作区根目录：

```bash
gh repo fork sethyuan/awesome-orcanote --clone -- awesome-orcanote
# 无 gh 命令时：提示用户在 GitHub 网页 fork，然后 git clone git@github.com:<用户名>/awesome-orcanote.git
```

**Step 5: 更新 plugins.json**

`awesome-orcanote/plugins.json` 是一个 JSON 数组，每项为一个插件对象。先按项目名称查找已有条目（匹配 `id` 或 `home` 字段）：

- **已存在**：更新 `version`（去掉 v 前缀）、`updated`（当前 UTC 时间 ISO 格式，如 `2026-01-01T03:00:00Z`）、`zip`（新版本下载链接），以及有变化的 `description`/`translations`
- **不存在**：新增条目，按 **先 author 后 id 的字母顺序**插入数组，条目结构：

```json
{
  "author": "<GitHub 用户名>",
  "id": "<插件 id，仅限字母数字连字符下划线>",
  "name": "<英文名称>",
  "description": "<英文描述>",
  "category": "<分类，如 Productivity / Integration / Theme>",
  "icon_svg": "<SVG 内容>",
  "version": "<x.x.x>",
  "updated": "<ISO UTC 时间>",
  "home": "https://github.com/<owner>/<repo>",
  "zip": "https://github.com/<owner>/<repo>/releases/download/v<x.x.x>/<zip>.zip",
  "translations": {
    "zh": { "name": "<中文名>", "description": "<中文描述>", "category": "<中文分类>" }
  }
}
```

图标字段二选一（推荐 SVG，尺寸 ≤80x80 px；PNG 建议压缩到 10KB 以内，照片类图先做 256 色量化），用仓库自带脚本生成内容：

```bash
node awesome-orcanote/scripts/minify-svg.js <path-to-svg>      # 生成 icon_svg 内容
node awesome-orcanote/scripts/png-to-base64.js <path-to-png>   # 生成 icon_png 内容
```

> ⚠️ 两个脚本的输出末尾会附加剪贴板提示语（如 `(Successfully copied to clipboard)`），重定向到文件时**只能取第一行的纯 base64/SVG 内容**，写入 plugins.json 前务必用严格模式校验（如 `python3 -c "import base64; base64.b64decode(open('x.txt').read().split(chr(10))[0], validate=True)"`），否则图标无法显示。

修改后用 `python3 -c "import json; json.load(open('awesome-orcanote/plugins.json'))"` 校验 JSON 合法性，然后在 awesome-orcanote 中提交（如 `feat: update <plugin-name> to <x.x.x>`）。推送到用户 fork 并向 sethyuan/awesome-orcanote 发起 PR 前，先向用户确认。

---

## 注意事项

- 所有 push、发 Release、提 PR 等外发操作必须先向用户确认
- 发布流程不操作 main 分支、不本地打 tag：开发分支 → main 的合并由用户在 GitHub 页面发起 PR，tag 随 GitHub Release 创建
- GitHub 直连（clone/pull）可能超时：SSH 失败回退 HTTPS；仍失败时使用本地已有副本并告知用户
- plugins.json 详细收录规范见 `awesome-orcanote/CONTRIBUTING.md`（id 字符限制、图标规范、zip 内容要求、排序规则）
- 版本号三处保持一致：git tag（`v` 前缀）、`package.json` 的 version、plugins.json 的 version（无 `v` 前缀）
- 插件仓库和发布包内的 `icon.png` 同样建议 ≤80x80 px：安装包体积更小，市场展示也一致
