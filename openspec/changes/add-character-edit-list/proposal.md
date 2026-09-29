## Why

用户打开右上角「角色」面板时，只能看到「＋ 新建角色」和「关闭」两个按钮，已存在的角色列表不显示，无法修改任何已有角色的信息。世界观面板能正常展示列表，角色面板也应如此。

代码层面，列表渲染函数 `renderCharList()`（index.html:963）开头有 `if ($("#charOverlay").hidden) return;` 守卫，而「角色」按钮的点击处理（index.html:1102）在 `openOverlay("charOverlay")` 打开面板**之前**先调用了 `renderCharList()`，此时面板仍处于 hidden 状态，守卫直接返回，列表永远不被填充。

## What Changes

- 修复角色面板的渲染时序/守卫逻辑：打开「角色」面板时必须渲染出当前世界观下已存在的角色列表（头像、姓名、设定摘要、出场开关、编辑/删除入口），与世界观面板的列表形态一致。
- 点入任一已存在角色（点击「编辑」按钮或角色信息区）即可打开预填好该角色信息的编辑表单，直接修改姓名、立绘、主题色、设定并保存。
- 保存、删除、切换出场状态后，角色列表与场景条、舞台区、视角下拉保持同步刷新（现有逻辑已具备，需确保修复后链路完整）。
- 不改变数据结构与预设内容；预置角色与世界观信息以 `preset/text.md` 为准（与 `seedStore()` 中的种子数据一致）。

## Capabilities

### New Capabilities

（无）

### Modified Capabilities

- `worldbuilding`: 「角色档案管理」补充面板级行为要求——角色管理面板打开时必须展示已存在角色列表，且每个角色可从列表直接进入编辑。（沿用 `cp-roleplay-chat` 变更中建立的 `worldbuilding` 能力路径；主 specs 目录当前为空，该变更尚未归档。）

## Impact

- 仅影响 `index.html` 单文件：角色面板打开时的事件处理顺序（`bindEvents` 中 `#charBtn` 的 onclick）与 `renderCharList()` 的守卫逻辑；可能涉及角色行信息区的点击绑定。
- 无数据结构、存储格式、API 变更；不影响已有 localStorage 数据、导入导出与世界观面板。
- 参考文档：`preset/text.md`（预设角色与世界观文案来源）。
