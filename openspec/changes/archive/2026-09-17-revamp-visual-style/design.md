# Design: revamp-visual-style

## Context

项目为单文件 `index.html`（内联 CSS/JS、零依赖、零构建，移动优先 `max-width: 480px`）。现有布局自上而下为：`header`（世界观 / 角色 / 设置）、`#sceneBar`（角色 chip 条）、`#chat`（对话区，`flex:1`）、`#inputBar`（视角下拉 + 输入框 + 发送/停止）；另有世界观、角色、设置三个 `.overlay` 弹层。已有可复用的基础：`hashColor(c.id)` 能从角色 id 派生稳定色相（现用于头像底色），角色 `avatar` 字段已是立绘图片 URL 的入口（留空即占位）。动机见 proposal.md - Why。

## Goals / Non-Goals

**Goals:**
- 用 CSS 自定义属性建立一套可整体切换的高饱和卡通主题，旧淡紫灰色值全部退场。
- 在 `#sceneBar` 与 `#chat` 之间插入舞台区组件，支持不透明立绘占位符、图片回退、出场联动、视角强调。
- 视角切换驱动背景主题平滑过渡，角色主题色无需新增数据字段。
- 交互动画（点击 / 聚焦 / 过渡）纯 CSS + 极少量 JS 实现，并支持 prefers-reduced-motion 降级。

**Non-Goals:**
- 不引入任何外部资源（字体、图片、动画库、框架），保持单文件零依赖。
- 不改 localStorage 数据结构、SSE 调用、HTML 消毒与消息渲染逻辑。
- 不做桌面宽屏重布局；不实现真实立绘资源的生产或 AI 生图。

## Decisions

### D1. CSS 变量驱动全局主题
在 `:root` 定义主题变量（主色 / 强调色 / 描边色 / 面板底色 / 文字色 / 阴影色等），全部旧色值替换为变量引用。卡通造型通过共享变量与工具类落地：粗描边（2–3px 实色边框）、大圆角 + 偏移硬阴影（`box-shadow: Npx Npx 0 色`）营造卡通立体感。
- **理由**：单文件内可维护性最好；视角主题切换（D3）等价于切换一组变量，无需 JS 逐元素改样式。
- **备选**：直接全量硬编码新色值——维护与视角切换都困难，拒绝。

### D2. 舞台区作为独立 flex 区块
新增 `#stage` 插在 `#sceneBar` 与 `#chat` 之间，`flex: none`、固定高度（约 150px 量级，实现时微调），横向排列立绘位、超出可横滑。立绘位（`.standee`）结构：底部为不透明角色主题色块（复用 `hashColor` 派生色）+ 角色名 + 像素风边框角标；有 `avatar` URL 时用 `<img>` 覆盖（`object-fit: cover`，`onerror` 移除回退占位，沿用现有 `avatarEl` 的模式）。占位符必须不透明、带角色名，满足 prototype 占位语义。当前视角角色的 `.standee` 加强调样式（放大 / 提亮 / 前置）；无出场角色时 `#stage` 隐藏（`hidden`），不挤占对话空间。
- **渲染挂载点**：`renderSceneBar` 与 `renderPovSelect` 已覆盖出场与视角变化，舞台渲染函数 `renderStage()` 由二者及 `renderAll` 调用即可覆盖全部联动路径。
- **备选**：立绘放在消息气泡旁——用户已选定顶部舞台区方案。

### D3. 视角主题 = hashColor 派生 + 容器级变量切换
每个角色视角的主题背景由其 `hashColor` 色相派生（同色相的浅色渐变 + 像素纹理叠加）；上帝视角用固定中性主题。切换方式：`renderPovSelect` / 视角 change 事件时在 `#app` 容器上写入 `--pov-*` 内联 CSS 变量（背景、强调色），背景属性挂 `transition` 实现平滑过渡。
- **理由**：复用现有派生色逻辑，零数据迁移；容器级变量一次写入、全局生效。
- **备选**：为角色新增「主题色」数据字段——破坏数据兼容、增加表单复杂度，拒绝。

### D4. 动画：CSS 为主，一个轻量 JS 涟漪
- 点击特效：`:active` 按压缩放（所有可点元素）+ pointer 事件动态生成涟漪 `<span>`（CSS keyframes 扩散后自移除）。
- 聚焦特效：`:focus`/`:focus-visible` 描边加粗变色 + 轻微脉冲 keyframes。
- 过渡：背景 / opacity / transform 上的 `transition`；弹层由 `hidden` 直接切换改为 class + transition（先显示再进场的两帧模式）；立绘位增删用进出场 keyframes。
- 流式更新路径（`updateStreamingBubble`）不加任何动画，动画属性限定在 `transform`/`opacity`/`background` 避免重排。
- **备选**：引入动画库——违反零依赖约束，拒绝。

### D5. prefers-reduced-motion 统一降级
单个 `@media (prefers-reduced-motion: reduce)` 块统一将动画 / 过渡压到近零时长；JS 涟漪生成前同样检查该媒体查询。

### D6. 像素点缀纯 CSS 实现
像素感来自：多层 `box-shadow` 拼像素块、阶梯式像素边框（分段 box-shadow 模拟）、棋盘格 / 点阵纹理（`repeating-conic-gradient`、`radial-gradient` 平铺）、像素风符号（▚▞◆✦ 等字符）。空态提示等区域加像素装饰。不加载外部字体或图片。

## Risks / Trade-offs

- [动画拖慢流式渲染或造成卡顿] → 流式路径不加动画；动画属性限定 `transform`/`opacity`/`background`；涟漪元素生命周期短且自动移除。
- [高饱和配色降低文字可读性] → 正文区保持深字浅底，高饱和色用于描边 / 按钮 / 装饰；实现后人工核对主要文本对比度。
- [舞台区挤占小屏对话空间] → 固定小高度 + 横向滚动；无出场角色时整体隐藏。
- [弹层从 `hidden` 改为 class 切换可能引入显示时序 bug] → 改动收敛在 `openOverlay`/`closeOverlay` 两个函数内，逐一回归三个弹层。
- [hashColor 派生的个别角色主题色可能偏灰或不美观] → 派生时固定较高饱和度与亮度区间（同现有 `hsl(h,55%,60%)` 思路调高）；接受小概率不完美，不为原型增加配置字段。

## Migration Plan

无数据迁移：localStorage 结构不变，老数据直接可用。部署即替换 `index.html`；回滚为还原该文件旧版本。
