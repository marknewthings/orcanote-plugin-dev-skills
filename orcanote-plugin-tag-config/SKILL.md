---
name: orcanote-plugin-tag-config
description: 开发「以标签为配置源」的虎鲸笔记（Orca Note）插件的架构模式与实现经验。核心思路：用户给块打上指定标签并填写标签属性，插件扫描标签块生成运行时配置（命令、快捷键、汇总表格）。涵盖标签自动创建与属性定义、扫描与属性提取、校验与无效原因、智能同步跳过、差量更新命令、自定义标签名与换标签迁移、启动时自动扫描等。当用户要开发让用户「打标签做配置」的 Orca 插件，或要给插件加标签扫描、标签属性读取、智能同步、开机自动扫描等能力时使用。
---

# 虎鲸笔记「标签即配置」插件模式

总结自 quick-jump（快捷跳转）与 super-walker（超级漫步）两个插件的共同架构：**用户在笔记里给块打标签、填标签属性，插件把标签块扫描成运行时配置**。适合做「用户标注一批块，插件为每个块生成命令/快捷键/行为」类型的插件。

## 模式总览

```
用户操作：给块打上 #插件标签，填写标签属性（名称、类型…）
                    │
插件同步（手动按钮 / 命令 / 启动自动扫描）
                    │
  ┌─ ensureTagExists：标签不存在则创建，补齐属性定义
  ├─ 扫描：get-blocks-with-tags 拉取全部标签块
  ├─ 提取：从 block.refs 中读取每个块的标签属性值
  ├─ 校验：必填、枚举、格式、重名 → valid / reason
  ├─ 智能同步：签名无变化则跳过（可开关）
  ├─ 差量更新：以 blockId 为主键 diff，注册/重注册/注销命令
  ├─ 汇总：在标签页生成配置汇总表格（可开关）
  └─ 持久化：configs / lastScan / lastTagId 存入插件数据区
```

模块划分建议（单向依赖 main → 功能模块 → orca 宿主）：
`settings.ts`（配置 schema + 常量）、`tag.ts`（标签自举）、`sync.ts`（扫描与差量更新）、`store.ts`(内存态+持久化)、`summary.ts`（汇总表格）、`restore.ts`（初次安装从汇总恢复）、`headbar.tsx`（顶栏同步按钮）。

---

## 一、标签自举（ensureTagExists）

启动和每次同步时确保标签存在且属性定义齐全，返回标签块 ID：

```ts
// 1. 按别名查标签块（标签名即别名）
const raw = await orca.invokeBackend("get-blockid-by-alias", tagName)

// 2. 不存在则创建：根级插入块 → 设置别名 → 设置图标
const newId = await orca.commands.invokeEditorCommand(
  "core.editor.insertBlock", null, null, null, [{ t: "t", v: tagName }])
await orca.commands.invokeEditorCommand(
  "core.editor.createAlias", null, tagName, newId)
await orca.commands.invokeEditorCommand(
  "core.editor.setProperties", null, [newId],
  [{ name: "_icon", type: 1, value: "ti ti-walk" }])  // _icon 是内置属性

// 3. 确保标签上有属性定义（缺失才补，避免重复 setProperties）
//    文本属性 type=1；单选属性 type=6 + typeArgs
const defs = []
if (!block?.properties?.some((p) => p.name === "名称")) {
  defs.push({ name: "名称", type: 1 })
}
const modeProp = block?.properties?.find((p) => p.name === "类型")
if (modeProp == null || modeProp.type !== 6) {   // 此前误建为文本类型也要重设
  defs.push({ name: "类型", type: 6, typeArgs: {
    subType: "single", choices: ["随机", "乱序"],
    defaultEnabled: true, default: "随机",
  }})
}
if (defs.length > 0) {
  await orca.commands.invokeEditorCommand(
    "core.editor.setProperties", null, [tagId], defs)
}
```

要点：
- 属性名、可选值全部用常量定义在 `settings.ts`，避免散落魔法字符串
- 单选属性（type=6）比自由文本更防错，能约束的枚举值尽量用单选
- 自动创建标签后 `orca.notify("success", ...)` 告知用户

## 二、扫描与属性提取

```ts
// 拉取所有打了标签的块（返回 Block[]，含 refs）
const blocks = await orca.invokeBackend("get-blocks-with-tags", [tagName])

// 必须排除标签块本身（标签页自身也在结果里）
const targets = blocks.filter((b) => Number(b.id) !== Number(tagId))

// 从块的标签引用中提取属性值：refs 中找到指向标签块的 ref，读 ref.data
function extractProp(block: Block, tagId: DbId, propName: string): string {
  const ref = block.refs.find(
    (r) => Number(r.to) === Number(tagId) && Array.isArray(r.data))
  for (const prop of ref?.data ?? []) {
    if (prop?.name === propName) {
      const v = prop.value
      if (typeof v === "string") return v.trim()
      if (Array.isArray(v)) return v.map(String).join("").trim()  // 单选值可能是数组
      if (v != null) return String(v).trim()
    }
  }
  return ""
}
```

坑：
- **属性值类型不定**：文本是 string，单选可能是数组，统一归一化成 trim 后的字符串
- **ID 比较必须 Number() 归一**：blockId 有时是 string 有时是 number
- 若插件只针对某类块（如查询块），扫描后校验块类型：`block.content?.[0]` 的 repr `type` 是否含 `"query"`

## 三、校验与无效原因（valid / reason）

每个标签块产出一条 `ScanEntry`，不合格的**不丢弃**，标记 `valid: false` + 中文 `reason`，最终写进汇总表格让用户自己看到哪里填错了：

```ts
interface ScanEntry {
  blockId: string
  name: string          // 显示名（可开启「块 ID 候补」：名称空时用 `块${blockId}`）
  valid: boolean
  reason?: string       // 标签属性有空 / 名称重复 / 格式错误 / 快捷键冲突…
}
```

校验顺序建议：必填项 → 枚举合法性 → 格式（如时长 `正整数+m/h/d`）→ 重名（先扫到的保留，`Map<name, blockId>`）→ 环境冲突（如快捷键被占用）。

## 四、智能同步（smartSync）

把「本次扫描的全部有效信息」序列化成签名，与上次持久化的签名相同就跳过更新：

```ts
const signature = JSON.stringify(entries.map((e) => [
  e.blockId, e.name, e.modeLabel, e.valid, e.reason ?? "",
  shortcutOf(e.blockId) ?? "",   // 外部可变状态也要纳入签名，
]))                              // 否则快捷键变了汇总不刷新
if (config.smartSync && lastScan === signature && !tagChanged) {
  if (!quiet) orca.notify("success", "同步完成：配置无变更", { title: "插件名" })
  return
}
```

要点：
- 签名要包含**所有会反映到产出（命令、汇总）的因素**，漏一项就会出现「变了但不刷新」
- 同步入口加 `syncing` 布尔锁防重入（`try/finally` 释放）
- `quiet` 参数：启动自动扫描时传 true，无变更不打扰用户

## 五、差量更新命令（以 blockId 为主键）

```ts
// prev = 上次配置，next = 本次有效条目
for (const [blockId, entry] of Object.entries(next)) {
  const old = prev[blockId]
  if (old == null) registerCommand(blockId, entry.name)          // 新增
  else if (old.name !== entry.name) {                            // 改名
    await unregisterCommand(blockId, /*clearShortcut*/ false)    // 命令 ID 不变
    registerCommand(blockId, entry.name)                         // → 快捷键保留
  }
}
for (const blockId of Object.keys(prev)) {
  if (next[blockId] == null) {
    // 关键：块还在扫描结果里（只是配置暂时填错）→ 保留快捷键绑定，
    // 用户修好属性后绑定自动恢复；彻底移除（摘标签/删块）才清绑定
    const stillTagged = keepIds.has(commandIdOf(blockId))
    await unregisterCommand(blockId, !stillTagged)
  }
}
```

命令 ID 用稳定映射（如 `${pluginName}.open.${blockId}`），改名不换 ID，快捷键绑定才能跨改名存活。

## 六、自定义标签名与换标签迁移

标签名做成插件设置项（`type: "string"`，有默认值），用户可随时改。改名后同步时要能检测并迁移：

```ts
// 持久化记录上次同步的标签块 ID
const lastTagId = await loadLastTagId(pluginName)
let tagChanged = lastTagId != null && Number(lastTagId) !== Number(tagId)
// 兜底：校验已记录的汇总块是否仍属于当前标签，不属于则删除旧汇总块
tagChanged = tagChanged || (await discardStaleSummaryBlocks(pluginName, tagId))
// tagChanged 时强制走完整同步（绕过 smartSync），在新标签下重建汇总
```

配置读取用「实时读」模式，设置修改立即生效、无需重启：

```ts
export function getConfig(pluginName: string) {
  const s = orca.state.plugins[pluginName]?.settings ?? {}
  const tagName = typeof s.tagName === "string" && s.tagName.trim() !== ""
    ? s.tagName.trim() : DEFAULT_TAG_NAME
  const smartSync = s.smartSync !== false        // 默认 true 用 !== false
  const autoScan = s.autoScanOnStartup === true  // 默认 false 用 === true
  // 数值项：Number() + isFinite 校验 + 范围夹取，非法回退默认值
  return { tagName, smartSync, autoScan, ... }
}
```

## 七、启动时自动扫描（开机启动）

设置项 `autoScanOnStartup`（默认关），`load()` 末尾延迟执行，等应用状态就绪：

```ts
if (getConfig(pluginName).autoScanOnStartup || conflicted) {
  setTimeout(() => {
    syncTags(pluginName, /*quiet*/ true).catch((err) =>
      console.warn("[plugin] 启动自动扫描失败:", err))
  }, 3000)   // 立即执行会因 orca.state 未就绪而拿不到块数据
}
```

`conflicted`：初次安装从旧汇总恢复配置时若发现快捷键冲突，也强制同步一次，把冲突原因写进汇总表格。

## 八、配套设置项清单（照抄即可）

| 设置项 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| tagName | string | 插件中文名 | 标记标签名，改后按新标签扫描 |
| showSyncButton | boolean | true | 顶栏同步按钮开关 |
| autoScanOnStartup | boolean | false | 启动时自动扫描（quiet 模式） |
| smartSync | boolean | true | 签名无变化跳过同步 |
| useIdFallback | boolean | true | 名称属性空时用块 ID 生成名称 |
| generateSummary | boolean | true | 在标签页生成配置汇总表格 |

## 踩坑速查

| 坑 | 对策 |
| --- | --- |
| 扫描结果包含标签块自身 | `filter((b) => Number(b.id) !== Number(tagId))` |
| 属性值时而 string 时而数组 | extractProp 统一归一化为字符串 |
| 智能同步漏刷新 | 快捷键、外部状态等一并纳入签名 |
| 改名导致快捷键丢失 | 命令 ID 含 blockId 不含名称；重注册不清绑定 |
| 配置暂时填错就清了快捷键 | 仍在扫描结果中的块保留绑定（keepIds） |
| 用户换标签后旧汇总残留 | lastTagId 比对 + 汇总块归属校验，删旧建新 |
| 启动立即扫描拿不到数据 | setTimeout 3 秒再同步 |
| 同步按钮连点重复执行 | 模块级 syncing 布尔锁 |
