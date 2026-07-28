# orcanote-plugin-dev-skills

开发[虎鲸笔记（Orca Note）](https://orcanote.com)插件的 Agent Skill，覆盖三大工作流：

1. **生成项目**：基于官方模板 [sethyuan/orca-plugin-template](https://github.com/sethyuan/orca-plugin-template) 创建插件项目骨架，附带 plugin-docs 离线 API 文档
2. **测试插件**：构建后将必需文件同步到本地 Orca 插件目录并重启 Orca Note
3. **发布版本**：开发分支递增版本号、构建发布包 zip、维护 [awesome-orcanote](https://github.com/sethyuan/awesome-orcanote) 插件市场注册表

技能内容见 [SKILL.md](SKILL.md)。

## 安装

Skill 采用通用的 `SKILL.md` 格式，将本目录复制到你的 Agent 技能目录即可：

```bash
# Qoder（个人技能，全局可用）
git clone https://github.com/<your-name>/orcanote-plugin-dev-skills.git \
  ~/.qoder/skills/orcanote-plugin-dev-skills

# Qoder（项目技能，随仓库共享）
git clone https://github.com/<your-name>/orcanote-plugin-dev-skills.git \
  <your-project>/.qoder/skills/orcanote-plugin-dev-skills

# Claude Code
git clone https://github.com/<your-name>/orcanote-plugin-dev-skills.git \
  ~/.claude/skills/orcanote-plugin-dev-skills
```

## 使用

安装后对 Agent 说：

- 「创建一个虎鲸笔记插件项目 my-plugin」→ 触发生成项目流程
- 「构建并安装到本地虎鲸笔记测试」→ 触发测试插件流程
- 「发布 my-plugin v0.2.0」→ 触发发布版本流程

## 说明

- 本地测试流程默认 macOS（`osascript` 重启应用）；其他平台手动重启即可
- 发布流程不操作 main 分支、不本地打 tag，合并与 Release 均由你在 GitHub 页面完成
- 所有 push / Release / PR 等外发操作，Agent 都会先向你确认

## License

MIT
