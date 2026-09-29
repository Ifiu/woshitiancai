## Context

`index.html` 单文件应用。角色管理面板（`#charOverlay`，index.html:266-281）的 DOM、`renderCharList()` 列表渲染（含头像/姓名/设定摘要/出场开关/编辑/删除按钮）、`openCharForm()` / `saveCharForm()` / `deleteChar()` 编辑保存链路均已存在。根因：`renderCharList()` 开头有 `if ($("#charOverlay").hidden) return;` 守卫，而 `bindEvents()` 中 `#charBtn` 的点击处理（index.html:1102）在 `openOverlay("charOverlay")` 之前先调用 `renderCharList()`，此时面板仍是 hidden，守卫直接返回，列表永远为空。世界观面板正常是因为 `renderWorldList()` 没有该守卫。

## Goals / Non-Goals

**Goals:**
- 打开角色面板即渲染已存在角色列表。
- 列表条目提供明确的编辑入口，点入后表单预填该角色当前信息。
- 修复后保存/删除/切换出场的同步刷新链路（场景条、舞台区、视角下拉）完整可用。

**Non-Goals:**
- 不改动数据结构、localStorage 存储格式与预设种子内容（预设文案以 `preset/text.md` 为准，现状已一致）。
- 不改动世界观面板、生成流程与导入导出。
- 不处理"生成中流式编辑角色"的并发语义（现状允许编辑，仅影响后续生成，维持不变）。

## Decisions

1. **在调用点修复时序，保留守卫**：把 `#charBtn` 的 onclick 改为先 `openOverlay("charOverlay")` 再 `renderCharList()`。
   - 备选 A：删掉 `renderCharList()` 的 hidden 守卫。也能修复，但守卫在 `toggleScene()` → `renderCharList()` 路径（面板未打开时）能避免无谓的 DOM 重建，属于有用的防御。
   - 备选 B：在 `openOverlay()` 里统一触发各面板渲染。改动面更大，且世界观面板沿用"调用点渲染"模式，为保持一致不引入新机制。
2. **条目信息区（`.grow`）也可点入编辑**：为角色行的 `.grow` 绑定 `openCharForm(c.id)`，与「编辑」按钮并存。
   - 理由：用户预期"点进去可以直接修改信息"；世界观面板条目同样是 `.grow` 可点（切换世界观）+ 独立「编辑」按钮的形态，交互一致。`.grow` 已在 ripple 效果的 `closest()` 选择器中，无需额外样式。
3. **同步链路不新增代码**：`saveCharForm()` / `deleteChar()` / `toggleScene()` 已调用 `renderCharList()` + `renderSceneBar()`（内部含 `renderStage()`）+ `renderPovSelect()`，修复渲染时序后该链路自然生效，仅需手动验证。

## Risks / Trade-offs

- [只改调用顺序，若未来新增其他打开角色面板的入口而忘记先 open 再 render，问题会复现] → 入口目前唯一（`#charBtn`）；在 tasks 验收中加入"打开面板即见列表"的回归检查。
- [角色列表在面板打开期间因流式生成完成等原因需要刷新] → 现状无此路径：`renderCharList()` 仅在面板交互（保存/删除/切换出场）时被调用，行为不变。
