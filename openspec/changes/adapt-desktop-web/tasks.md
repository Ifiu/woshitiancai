## 1. 宽屏布局断点

- [x] 1.1 在 `index.html` 的 `<style>` 末尾（`prefers-reduced-motion` 块之后）新增 `@media (min-width: 768px)` 块：`#app { max-width: 640px; }`、`#toast { max-width: 480px; }`。验证：浏览器窗口调至 800px 宽，主容器约 640px 且水平居中，toast 不再按 86vw 拉伸。
- [x] 1.2 追加 `@media (min-width: 1024px)` 块：`#app { max-width: 860px; }`。验证：窗口调至 1440px 宽，主容器 860px 居中，两侧露出页面背景。
- [x] 1.3 在 ≥768px 块内改写弹层形态：`.overlay { align-items: center; padding: 24px; }`；`.sheet { max-width: 560px; border-bottom: 3px solid var(--ink); border-radius: 20px; max-height: 80vh; padding-bottom: 16px; transform: translateY(12px); }`。验证：桌面端分别打开世界观 / 角色 / 设置弹层均为居中对话框、超高内容可滚动；375px 视口下三者仍为底部抽屉。

## 2. 舞台与阅读尺寸适配

- [x] 2.1 在 ≥1024px 块内放大舞台：`#stage { height: 210px; }`、`.standee { width: 116px; height: 170px; }`、`.standee .nm { font-size: 12px; }`。验证：桌面端舞台更高、立绘更大，角色出场 / 离场动画与当前视角高亮（`.standee.focus`）表现不变。
- [x] 2.2 在 ≥1024px 块内限制对话宽度：`.msg { max-width: 78%; }`、`.msg.assistant { max-width: 84%; }`。验证：桌面端发送一条长提示词，AI 长回复气泡不横跨整个容器，用户气泡仍靠右、AI 气泡靠左。

## 3. 键鼠交互优化

- [x] 3.1 新增独立 `@media (hover: hover) and (pointer: fine)` 块，为 `header .tool, .btn, .chip, #sendBtn, #stopBtn, .msg-actions button, .sheet .row .grow, #worldBtn` 增加 hover 态（仅 `transform: translateY(-1px)`、`box-shadow` 或底色微调，不改盒尺寸）。验证：桌面端鼠标悬停各元素有可见反馈、移开恢复、点击时下沉态仍正常；系统开启「减弱动态效果」时位移消失。
- [x] 3.2 在 ≥768px 块内为 `#chat, #sceneBar, #stage, .sheet` 双写滚动条样式：`scrollbar-width: thin; scrollbar-color: var(--ink-soft) transparent;` 及 `::-webkit-scrollbar`（纵向宽 8px / 横向高 6px，thumb 圆角、`var(--ink-soft)`，hover 转 `var(--ink)`，track 透明）。验证：桌面 Chrome 与 Firefox 下聊天区滚动条为细主题样式；对话很少无溢出时不显示滚动条轨道。
- [x] 3.3 在 `bindEvents()` 中为 `#promptInput` 增加 `keydown` 处理：`e.key === "Enter"` 且未按 Shift、`!e.isComposing`、`e.keyCode !== 229`、且 `matchMedia("(pointer: fine)").matches` 为真时 `e.preventDefault()` 并调用 `send()`。验证：桌面端 Enter 发送、Shift+Enter 换行、中文输入法组词中按 Enter 仅上屏不发送、流式生成期间按 Enter 不重复发送；手机端回车仅换行。

## 4. 回归与收尾

- [x] 4.1 移动端回归：375px（或手机真机）视口逐项对照——主容器 480px、弹层底部抽屉、舞台 150px、气泡 92%、无 hover 残留、回车仅换行，全部与适配前一致。
- [ ] 4.2 桌面端（1440px）端到端走查：发送→流式输出→停止→重新生成→删除→清空，切换视角主题色，三个弹层的新建 / 编辑 / 删除 / 开关出场，导出 / 导入 JSON，均正常可用。
- [x] 4.3 更新 `README.md` 中「手机端单页」「手机上访问更佳」等表述为「移动端 + 桌面网页端均适配」，与实际行为一致。
