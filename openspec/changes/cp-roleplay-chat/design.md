## Context

全新空仓库，目标是手机浏览器里可用的单页文字游戏。用户无后端、无账号体系；LLM 能力来自用户自备的 OpenAI 兼容 API。用户已确认的增量决策：需要 SSE 流式输出；生成参数先固定默认值；初始角色"张极""左航"与初始世界观用占位内容（用户有设定，后续自行替换）；不做 Markdown，AI 回复中合适时直接输出 HTML+CSS 富渲染。

## Goals / Non-Goals

**Goals:**
- 零构建、零后端的静态单页应用，手机浏览器直接可用。
- 世界观作为顶层数据容器，切换是纯粹的状态切换，成本为 O(1) 且无数据风险。
- Prompt 组装逻辑清晰可维护：世界观 + 出场角色 + 视角 + 历史 → 一次流式 Chat Completions 调用。
- SSE 流式渲染：内容随到达逐段显示，可中止。
- 受约束 HTML 富渲染：让 AI 能输出剧情化的富元素（聊天界面卡片、回忆滤镜框、系统提示等），同时保证安全。

**Non-Goals:**
- 不做图片/语音生成、不做多设备同步、不做用户账号、不做 Markdown 渲染。
- 不内置任何 API Key 或默认模型服务商。
- 不向用户暴露生成参数调节（先固定默认值，后续再议）。
- 不做内容审核层（用户自用的私人工具）。

## Decisions

### D1: 纯静态单文件应用（index.html 内联 CSS/JS）
手机端私人小工具，引入框架和构建链（React/Vite）收益低、门槛高。单文件可直接双击打开或丢到任意静态托管。
- 备选：React + Vite → 拒绝：需要 Node 环境和构建步骤，违背"即开即用"。

### D2: 世界观切换 = 切换 currentWorldId
数据模型以世界观为顶层容器：

```
store = {
  v: 1,
  settings: { apiBase, apiKey, model },
  currentWorldId: string,
  worlds: {
    [worldId]: {
      id, name, setting,            // 世界观设定自由文本
      characters: [ { id, name, profile, avatar? } ],  // avatar 为图片 URL，缺省用占位符
      scene: [characterId],         // 当前出场角色
      messages: [ { id, role, content, pov, ts } ]
    }
  }
}
```

切换世界观只改 `currentWorldId`，整个 UI（顶栏、角色列表、对话区）由当前世界观派生渲染。顶栏放世界观名下拉，点开即切换列表 + "新建世界观"。首次启动（store 为空）时播种：一个占位世界观 + "张极""左航"两个角色（占位档案、无 avatar 走占位符）。
- 备选 A：多世界观并排标签页 → 拒绝：手机屏幕宽度不够，且无同时看两套的需求。
- 备选 B：世界观作为对话内的"频道" → 拒绝：会让对话记录跨世界观混杂，违背数据隔离要求。

### D3: 视角作为每条消息的元数据 + prompt 指令
发送时用户选视角（上帝 / 出场角色之一）。组装 system prompt 时注入视角指令：角色视角要求"以 X 的第一人称叙述，包含其内心活动，不暴露他人内心"；上帝视角要求"第三人称全知叙述"。AI 回复消息记录所用视角，便于回溯。

### D4: Prompt 组装与历史窗口
单次请求的消息序列：
1. system：世界观设定 + 出场角色档案（姓名+设定）+ 视角指令 + 输出格式约定。输出格式约定包含两部分：
   - 默认剧本式：`【角色名】动作/台词`，允许叙述段落；
   - 富元素约定：当剧情出现合适元素（如手机聊天界面、回忆片段、系统/旁白提示）时，允许直接输出 HTML 片段，且只能用内联样式，并明确告知白名单标签集合，引导模型不输出 script/事件属性。
2. 最近 N 条历史（默认 20，超出截断最旧的，防 token 膨胀）。
3. user：用户提示词。
- 备选：把角色档案塞每条 user 消息 → 拒绝：重复浪费 token 且语义混乱。

### D5: localStorage 单 key 存整个 store JSON
键 `cp-rp-store`，每次变更整体序列化写入。导出/导入即该 JSON 的下载与读取（版本字段 `v: 1`，为将来迁移留口）。
- 备选：IndexedDB → 拒绝：本场景数据量小（纯文本），localStorage 足够且实现简单。

### D6: SSE 流式调用
浏览器 `fetch` POST `{apiBase}/chat/completions`，body 带 `stream: true`，Bearer 鉴权。用 `response.body.getReader()` 读取字节流，按 SSE 协议解析 `data:` 行（兼容 OpenAI 的 `data: [DONE]` 结束标记与 `choices[0].delta.content` 增量结构），每收到增量即追加到当前气泡并触发渲染。AbortController 实现"停止生成"与整体超时；停止时已收到部分落盘为完整消息。
- 备选：EventSource → 拒绝：EventSource 只支持 GET 且不能自定义请求头，无法做 Bearer 鉴权的 POST。
- 备选：非流式一次性返回 → 拒绝：用户明确要求 SSE，且长回复等待感差。

### D7: 受约束 HTML 渲染（替代 Markdown）
AI 气泡内容分两类：纯文本按 textContent 渲染（保留换行）；检测到包含 HTML 标签时走"消毒后 innerHTML"路径。消毒用自实现的轻量白名单过滤（DOMParser 解析 → 遍历节点 → 仅放行白名单标签如 `div/span/p/b/i/em/strong/br/hr/img(限 https data:)`，属性仅放行 `style` 与 `img` 的 `src/alt`，style 内剔除 `url(`/`expression` 等危险片段；其余节点 unwrap 或剔除）。不引入第三方库，保持单文件零依赖。
- 备选 A：Markdown 渲染 → 拒绝：用户明确不需要，且表达力弱于 HTML。
- 备选 B：DOMPurify 等库 → 拒绝：破坏单文件零依赖原则；白名单范围小，自实现可控。
- 风险：消毒器永远不可能完美 → 见 Risks。

### D8: 固定生成参数
请求体固定一组默认值：`temperature: 0.9`、`top_p: 0.95`、`max_tokens: 2048`（角色扮演偏创造性，取值偏高）。常量集中在代码顶部，后续要开放调节时只需加设置项。

### D9: 立绘占位符
角色无 `avatar` 时，头像渲染为圆形色块（颜色由角色 id 哈希决定）+ 名字首字。设置 avatar（图片 URL）后改显示图片。用户后续把"张极""左航"的立绘原型替换进来只需编辑角色档案。

## Risks / Trade-offs

- [HTML 消毒存在绕过风险（如冷门属性、CSS 注入）] → 白名单从严：只允许少量标签与 `style` 属性且过滤危险 CSS；应用不引入第三方脚本，即使绕过也无敏感接口可调；API Key 风险在设置页明示。
- [localStorage 约 5MB 上限，长期聊天记录可能撑爆] → 提供一键导出 JSON 备份；写入失败时提示用户导出并清理。
- [API Key 明文存 localStorage，共用设备有泄漏风险] → 设置页明示风险提示。
- [部分 API 端点不允许浏览器跨域调用（CORS）或 stream 被中间层缓冲] → README 建议选择支持 CORS 与流式的端点/中转；错误提示中给出可能原因；前端解析对流中断有兜底（已收部分保留）。
- [流式渲染中 HTML 标签未闭合导致渲染闪烁] → 流式阶段按纯文本渲染增量，流结束后再走一次完整 HTML 消毒渲染（两阶段渲染）。
- [无构建工具意味着无自动化测试框架] → 关键纯函数（prompt 组装、SSE 解析、HTML 消毒、store 读写）保持无 DOM 依赖，可在浏览器控制台手动验证；验收以 spec 场景手工走查为准。

## Migration Plan

全新项目，无迁移。交付即仓库根目录 `index.html`；本地打开或任意静态托管即可使用。回滚 = 删除文件。

## Open Questions

- 生成参数后续是否开放调节、开放哪些（当前固定默认值即可，不影响本次实现）。
