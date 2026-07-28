# orcanote-plugin-dev-skills

虎鲸笔记（[Orca Note](https://orca.so)）插件开发的 Agent Skills 集合，可用于 Qoder、Claude Code 等支持 Agent Skill 的 AI 编码工具。

## 包含的技能

| 技能 | 说明 |
| --- | --- |
| [orcanote-plugin-dev-steps](./orcanote-plugin-dev-steps/SKILL.md) | 插件开发三大流程：生成项目（基于官方模板）、测试插件（同步到本地 Orca 并重启）、发布版本（打 zip、维护 awesome-orcanote 插件市场注册表） |
| [orcanote-plugin-tag-config](./orcanote-plugin-tag-config/SKILL.md) | 「标签即配置」插件架构模式：用户给块打标签+填标签属性，插件扫描生成命令/快捷键/汇总，涵盖标签自举、属性提取、智能同步、差量更新、换标签迁移等实现经验 |

## 安装

把需要的技能目录放到你的 skills 目录下即可：

```bash
git clone https://github.com/marknewthings/orcanote-plugin-dev-skills.git

# Qoder 全局技能
cp -R orcanote-plugin-dev-skills/orcanote-plugin-* ~/.qoder/skills/

# 或仅当前项目
cp -R orcanote-plugin-dev-skills/orcanote-plugin-* <你的项目>/.qoder/skills/

# Claude Code
cp -R orcanote-plugin-dev-skills/orcanote-plugin-* ~/.claude/skills/
```

## 使用

安装后直接向 AI 提需求即可自动触发，例如：

- 「帮我创建一个虎鲸笔记插件项目，名叫 my-plugin」（dev-steps · 生成项目）
- 「构建并安装到本地虎鲸笔记测试一下」（dev-steps · 测试插件）
- 「发布版本 v0.2.0」（dev-steps · 发布版本）
- 「我要做一个通过打标签来配置书签的插件」（tag-config）

## 说明

- 流程中的本地插件目录默认按 macOS 路径（`~/Documents/orca/plugins/`）举例，其他平台以 Orca Note 设置中的数据目录为准
- 发布流程默认「开发分支提交 + GitHub PR 合并 + Release 挂 zip」的协作方式，可按自己的习惯调整

## License

[MIT](./LICENSE)
