## Context

`index.html` 单文件应用，全部样式集中在头部 `<style>`（index.html:7-229）。与本次适配相关的现状：`#app` 与 `.sheet` 均 `max-width: 480px`（index.html:33, 180）；`.overlay` 为底部抽屉布局（`align-items: flex-end`，index.html:172-176）；`#stage` 高 150px、`.standee` 84×124px（index.html:79-95）；`.msg` 限宽 92%、`.msg.assistant` 全宽（index.html:125-127）；`#toast` 限宽 `86vw`（index.html:210）；按钮仅有 `:active` 态、无任何 `:hover` 与滚动条样式。交互侧：发送仅靠 `$("#sendBtn").onclick = send`（index.html:1100），输入框无 `keydown` 处理；`send()` 已内置守卫（流式中 / 空输入直接 return，index.html:583-587），Enter 复用它即与按钮完全等价。已有 `@media (prefers-reduced-motion: reduce)` 全局降级（index.html:223-228）。

## Goals / Non-Goals

**Goals:**
- 宽屏分档放宽主容器，桌面端达到舒适阅读宽度，移动端像素级不变。
- 桌面端弹层、舞台、气泡、toast 尺寸符合大屏使用预期。
- 键鼠设备获得 hover 反馈、主题滚动条、Enter 发送（含 IME 保护）。

**Non-Goals:**
- 不改任何 ≤767px 下的既有 CSS 规则与 JS 行为（宽屏样式全部写在新增媒体查询块内，不触碰基础规则）。
- 不引入 CSS 框架 / 构建工具 / 新依赖，不调整 viewport meta。
- 不做多栏布局、不做窗口拖拽缩放的连续流式尺寸（用固定档位）。
- 不改动 `send()`、流式生成、渲染与存储逻辑（Enter 仅复用现有入口）。

## Decisions

1. **两档 `min-width` 断点，样式只增不改**：新增 `@media (min-width: 768px)`（`#app` max-width → 640px）与 `@media (min-width: 1024px)`（→ 860px）两个块，所有桌面规则写进块内覆盖，基础规则一行不动。
   - 备选 A：`clamp()` 连续缩放。被否——舞台高度、立绘尺寸、气泡比例需要档位化的离散调整才好看，连续缩放反而处处要设中间值。
   - 备选 B：只设一档 ≥1024px。被否——平板（768–1023px）会退回 480px 窄条，与桌面端是同一类问题。
2. **弹层在 ≥768px 变为居中对话框**：`@media (min-width: 768px)` 内 `.overlay { align-items: center; padding: 24px; }`；`.sheet { max-width: 560px; border-bottom: 3px solid var(--ink); border-radius: 20px; max-height: 80vh; padding-bottom: 16px; }`，入场位移从 40px 收小到 12px。遮罩、开关逻辑、DOM 结构均不动。
3. **舞台与阅读尺寸仅在 ≥1024px 放大**：`#stage` 高 → 210px，`.standee` → 116×170px，`.standee .nm` 字号 11→12px；`.msg` → `max-width: 78%`，`.msg.assistant` → `max-width: 84%`（约 720px 行长上限）；`#toast { max-width: 480px }`（此条放 ≥768px 档）。立绘对齐方式保持左对齐横滚，不改交互。
4. **hover 反馈挂在能力查询而非宽度断点**：独立 `@media (hover: hover) and (pointer: fine)` 块，为 `header .tool, .btn, .chip, #sendBtn, #stopBtn, .msg-actions button, .sheet .row .grow, #worldBtn` 增加 hover 态（`transform: translateY(-1px)` + 阴影 / 底色微调，与既有 `:active` 下沉态衔接）。
   - 理由：宽度与是否键鼠无必然关系（触屏笔记本、手机横屏都 ≥768px）；`hover: hover` 查询天然排除触屏残留样式，且被 `prefers-reduced-motion` 全局降级覆盖。
5. **主题滚动条放 ≥768px 档**：对 `#chat, #sceneBar, #stage, .sheet` 双写标准属性（`scrollbar-width: thin; scrollbar-color: var(--ink-soft) transparent;`）与 `::-webkit-scrollbar` 家族（纵向 8px / 横向 6px，thumb 圆角 + `--ink-soft`，hover 转 `--ink`，track 透明）。
   - 备选：全局生效。被否——虽然移动端覆盖式滚动条基本不显示轨道，但放进断点内能把移动端回归面降到绝对零。
6. **Enter 发送挂在输入框 `keydown`，复用 `send()`**：`bindEvents()` 中新增——`if (e.key === "Enter" && !e.shiftKey && !e.isComposing && e.keyCode !== 229 && matchMedia("(pointer: fine)").matches) { e.preventDefault(); send(); }`。
   - 判定用 `pointer: fine` 而非宽度：桌面浏览器窗口拉窄时用户仍预期 Enter 发送；触屏设备（`pointer: coarse`）自动保持纯换行。事件触发时实时查询，无需监听断点变化。
   - IME 双保险：`isComposing` 覆盖现代浏览器，`keyCode === 229` 兜底旧 WebKit 组词上屏。
   - 直接调 `send()` 而非 `$("#sendBtn").click()`：`send()` 已有流式中 / 空输入守卫（index.html:584-587），按钮本身也只是调 `send()`，两者完全等价；Shift+Enter 不满足条件，自然走默认换行。

## Risks / Trade-offs

- [某条桌面规则误写进基础样式导致手机端回归] → 约定所有新规则只进媒体查询块；验收时用 375px 与 1440px 两档视口对照检查。
- [`pointer: fine` 在带键盘的平板上判定为 coarse，Enter 不发送] → 属可接受的保守降级（仍可按发送按钮），规格中已按"宽屏或精确指针"表述。
- [hover 位移造成相邻元素抖动] → hover 态只动 `transform` 与 `box-shadow`，不改盒模型尺寸。
- [滚动条样式 Firefox / WebKit 语法互不识别] → 双写两套属性，各自生效，无依赖关系。

## Migration Plan

纯静态单文件，替换 `index.html` 即完成发布；无数据格式与存储变更，回滚即还原文件。
