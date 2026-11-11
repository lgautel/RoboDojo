# 阿里千问浏览器 Agent：深度技术调研与多方案系统设计

> 文档编号：`cua_g4c_1`  
> 成文日期：2026-11-11  
> 文档性质：自包含的技术调研与架构设计。只描述系统，不提供项目实现源码。  
> 硬件基线：Windows 11，无独显，AVX2 纯 CPU；系统内存 16 GB；扣除系统与常驻 Chrome 后，编排与推理的安全可用物理内存按 6–8 GB 计；推理常驻工作集硬顶 **4.8 GB**。  
> 业务对象：用户本人已登录的国内版千问网页 `https://www.qianwen.com`。  
> 最高工程指标：个人主号不被封禁、不被限流、不被写入风控黑名单。  
> 证据边界：页面选择器、流式字段与风控组件名称，来自 2025-11 至 2026-11 的公开适配器、前端脚本与安全社区对阿里 Web 纵深防御的逆向观察。CSS Modules 哈希类名会随发版变化，正文把它们写成**契约锚点**，实现时必须以语义属性复验，不能把哈希类名当成永久 API。

---

## 第 0 章 执行摘要与核心反直觉洞察

### 0.1 一句话裁决

这个系统的大脑不该是“每一步都让 4B 小模型看整页 DOM 再决定点哪里”的 ReAct 循环。正确的形态是一台**站点确定性状态机**：它用人类可信输入把问题送进已登录的千问，用与页面脚本隔离的只读观察把“深度思考”和“正式回复”拆开拿走，再用同一台纯 CPU 上的 4B 模型做两件低频的事——页面异常时的语义自愈，以及拿千问自己的思考链审计它的正式回复。浏览器执行面准备五套可切换方案，默认主号走读写物理分离，日常后台可切到 BrowserSkill，协议级原文需求再打开受约束的流旁路。

### 0.2 四条必须先接受的工程洞察

**洞察一：快慢双轨，慢轨默认睡着。**  
在 AVX2 纯 CPU 上，`Qwen3.5-4B` Q4 的解码大约 15–24 token/s，一次“看快照、想动作、出 JSON”要 15–40 秒。千问页面只有十几个稳定控件：深度思考开关、输入框、发送、停止、思考折叠区、正文区、复制。把这些控件做成带版本号的微技能，命中时零模型调用。只有选择器失效、出现登录框或 `baxia-dialog`、或完成态三信号互相矛盾时，才唤醒本地模型做一次语义定位，并把新选择器写回缓存。这就是 Stagehand V3 `ActCache` 的思想：缓存动作序列，失败后再快照、再推理、成功则覆盖缓存。差别是本系统的快路径连浏览器驱动都不经过通用 Agent 框架，模型只在自愈与审计两处出现。

**洞察二：读和写必须物理分离。**  
阿里前端同时看三样东西：调试器是否附着、原生函数是否被改过、输入事件是否 `isTrusted === true`。只读用扩展的 `ISOLATED` 世界（或完全在浏览器外的 UIA），写用操作系统输入队列或 USB HID。三条暴露面同时消失：没有 `chrome.debugger`，没有主世界 `fetch` 包装，点击和按键对页面来说是可信事件。合成事件、主世界 Hook、长期挂着的调试横幅，三条里占两条就足以让个人主号不值得冒这个险。

**洞察三：思考和正文是两条通道，不是一块 `innerText`。**  
国内站深度思考在协议里活在 `multi_load` 中 `type === "deep_think"` 的 `content.think_content`，正文活在主 `content`，中间用 `[(deep_think:...)]` 这类占位符衔接。视图上则是顶部折叠区和 `#qk-markdown-react` 正文。对整个回答容器取文本，展开时混采，折叠时漏采。采集器必须是双区状态机，完成态看三个独立信号：发送按钮回到可发送、思考标题从“思考中”变为“已深度思考（用时 X 秒）”、两个区的文本在静默窗口内不再增长。

**洞察四：4B 评不了 300B 的世界知识，但评得了旗舰是否背叛了自己的思考。**  
小模型当全知裁判，会把旗舰写对的冷门事实判成幻觉，也会被排版和自信语气放过真正的错误。本系统把评价改成闭卷审计：Python 先做长度、代码围栏、禁用词等确定性检查；模型按固定因果顺序先摘录思考、再摘录正文、再写短批判、最后给 `PASS` / `FAIL` / `NEEDS_EXTERNAL_VERIFY`。五维只覆盖思考链和正文之间的忠实度。凡是判失败却摘不出原文子串的指控，由 Python 丢弃。

### 0.3 五套执行面与推荐路线

| 方案 | 一句话 | 主号默认？ | 焦点 | 思考/正文完整度 |
| --- | --- | --- | --- | --- |
| 一、读写物理分离 | 无 `debugger` 的 `ISOLATED` 扩展只读 + `SendInput`，可升级 ESP32-S3 / RP2040 USB HID | 是，安全天花板 | 占用前台 | DOM 双区高，协议原文无 |
| 二、BrowserSkill 后台 | `bsk` + 独立 Agent Window + `cooperate sync` | 日常后台首选 | 不抢焦点 | 可见文本中等，代码围栏可能降级 |
| 三、Patchright 专属 Profile | 商业版有头 Chrome + 独立目录，首次扫码养号 | 否，无人值守演进 | 独立窗口 | DOM 高，可听 CDP 帧 |
| 四、安全流旁路 | 真实输入不变；默认在生成结束后做一次同源只读对账，流式分流默认关闭 | 仅当必须要原始 `deep_think` JSON | 随输入通道 | 协议级无损 |
| 五、纯 OS 外围 | 零扩展、零端口，UIA 几何 + `SendInput` + 原生复制 | 极客基线 / 对照 | 独占前台 | 低到中，复制结果经常分不开两栏 |

演进顺序固定为三段，不并行铺开五套实现：

1. **验证期（方案五或方案一的只读半边）**：先证明双区文本能分开、完成态不会误判、未知提交不会重发。方案五零安装，适合第一天对照；一旦复制按钮拿不到折叠中的思考全文，立即切到方案一的只读扩展。
2. **日常期（方案一，或把读改挂到方案二）**：主号输入走可信事件。需要挂机不抢鼠标时，读和点击改走方案二的 Agent Window，节流器与状态机不变。
3. **协议期与硬件期**：确认 DOM 会丢掉公式源码或工具调用卡片后，才启用方案四的完成后单次同源拉取；确认 `SendInput` 的注入标志成为实际风险后，再把写通道换成 USB HID。方案三只在“必须无人值守且不能碰日常 Chrome”时单开一个养过的 Profile，不拿它操作日常主号。

### 0.4 数值锁定（后文不得另起一套）

| 项 | 锁定值 |
| --- | --- |
| 推理工作集硬顶 | 4.8 GB |
| 默认评审模型 | `Qwen3.5-4B`，GGUF `Q4_K_M`，8K 上下文下进程约 3.1–3.4 GB，解码 15–24 token/s |
| 指令更稳的平替 | `Qwen3-4B-Instruct-2507`，`Q4_K_M` 约 2.50 GB 文件，8K 下约 3.3 GB，解码 17–29 token/s |
| 越级裁判 | `CompassJudger-2-7B-Instruct` 的 `Q4_K_M` / `IQ4_XS`，8K 下约 5.1–5.5 GB。**超过 4.8 GB 硬顶**，只有空闲物理内存稳定 ≥ 7.5 GB 且用户显式抬高水位线时才允许加载 |
| 明确淘汰 | `Qwen3-30B-A3B`（`Q4_K_M` 约 18.6 GB）；任何 `enable_thinking=true` 的小模型 |
| 本地单次审计输出 | 预算 600 token，典型 250–400 token，约 12–25 秒 |
| 慢轨自愈 | 每次任务至多 2 次模型调用；第 2 次仍失败即人工接管 |
| 站点并发 | 全局 1。不允许两个方案同时向同一账号发消息 |

### 0.5 文档地图

| 章 | 回答的问题 |
| --- | --- |
| 1 | 页面上到底有什么，协议里思考和正文怎么拆 |
| 2 | 阿里风控看什么，主号哪十条不能破 |
| 3 | 点名的 8 个产品和扩大的技术里，什么能当底座 |
| 4 | 16 GB 纯 CPU 跑哪个模型，怎样做弱评强 |
| 5 | 五套执行面各自的数据流和暴露面 |
| 6 | 七维对比和三段落地 |
| 7 | 与方案解耦的状态机、采集器、节律器、UI、时延和恢复 |

---

## 第 1 章 需求拆解与 `qianwen.com` 页面 / 协议解剖

### 1.1 六步闭环

系统只做这一条闭环，不做通用上网代理。

| 步骤 | 动作 | 成功判据 | 失败时 |
| --- | --- | --- | --- |
| 1 | 本地 UI 提交一条问题，附带是否开启深度思考 | 任务落盘，状态 `QUEUED` | UI 拒绝空问题 |
| 2 | 编排器确认浏览器里是已登录的 `www.qianwen.com` 会话页 | URL 为 `/chat/<32 hex>` 或可新建会话的首页，且无登录框 | 停在 `NEED_LOGIN`，不自动登录 |
| 3 | 按人类节律把问题写入作曲器并发送 | 发送后按钮进入停止态，或回答区出现新的 `data-req-id` | 见 1.2，禁止盲重发 |
| 4 | 分离采集思考区与正文区，直到完成态 | 三重完成信号一致，两段文本分别落盘 | 超时则 `PARTIAL`，保留已采文本 |
| 5 | 本地模型按第 4 章审计 | 产出通过子串回验的 JSON 报告 | 模型超时则报告标 `JUDGE_UNAVAILABLE`，原始回复仍展示 |
| 6 | UI 三栏展示思考、正文、审计 | 三栏指向同一 `task_id` | 允许只展示已采到的栏 |

```mermaid
stateDiagram-v2
    [*] --> QUEUED
    QUEUED --> ARMED: 浏览器就绪且深度思考开关符合预期
    QUEUED --> NEED_LOGIN: 登录框或游客试聊耗尽
    ARMED --> TYPING: 输入框聚焦且包围盒有效
    TYPING --> SUBMIT_UNKNOWN: 已发出确认动作
    SUBMIT_UNKNOWN --> STREAMING: 出现新回答锚点或停止按钮
    SUBMIT_UNKNOWN --> NEED_HUMAN: 滑块或处罚页
    SUBMIT_UNKNOWN --> RECONCILE: 静默超时
    RECONCILE --> STREAMING: 对账发现新消息
    RECONCILE --> SAFE_RETRY: 对账确认未发送
    SAFE_RETRY --> TYPING: 仅此一次
    STREAMING --> CAPTURED: 三重完成
    STREAMING --> PARTIAL: 墙钟超时
    CAPTURED --> JUDGING
    PARTIAL --> JUDGING
    JUDGING --> SHOWN
    NEED_LOGIN --> [*]
    NEED_HUMAN --> [*]
    SHOWN --> [*]
```

`NEED_HUMAN` 包含 `baxia-dialog`、`punish`、`RGV587_ERROR` 页面、验证码和任何本系统不认识的模态框。进入该状态后，编排器停止一切输入，UI 发声提示，人处理完再手动点继续。系统不尝试自动过滑块。

### 1.2 未知提交的幂等边界

千问一次发送会消耗账号配额并留下服务端会话。最危险的故障是“点击已经生效，但本地没看到新节点”，然后状态机再点一次发送。

规则：

1. 进入 `TYPING` 之前，记录当前最后一个回答节点的 `data-req-id` 或 `data-msgid`，记为 `anchor_before`。
2. 确认动作（点击发送，或作曲器上的 Enter）只发出一次。动作发出的瞬间状态改为 `SUBMIT_UNKNOWN`，并立刻把 `{task_id, anchor_before, question_hash, wall_time}` 写入磁盘。进程崩溃重启后若发现该记录，禁止再发送，只能对账。
3. 对账顺序与第 7.2 节的守卫相同：出现新的回答锚点或停止按钮，就进入 `STREAMING`。两者都没有，并且作曲器里仍能看见问题文本的前 20 个字，才允许 `SAFE_RETRY` 一次。作曲器已被清空但又没有新回答，视为状态不明，进入 `NEED_HUMAN`，不重发。
4. 一次任务的发送次数上限是 2（初次 + 一次安全重试）。第二次仍未知，进入 `NEED_HUMAN`。
5. 问题文本的哈希随任务持久化。重试时必须仍是同一哈希，禁止用户在 `SUBMIT_UNKNOWN` 期间改题后被旧状态机发出去。

### 1.3 三个容易混用的站点

| 站点 | 2026 年的角色 | 本系统 |
| --- | --- | --- |
| `www.qianwen.com` | 国内 C 端主站。2025-11-14 通义品牌升级为千问后的对话入口 | 唯一目标 |
| `tongyi.aliyun.com` | 实验室展示门户。`/qianwen`、`/chat` 301 到 `www.qianwen.com`；旧 `conversation` v1 已下线 | 只跟随 301，不适配旧 DOM |
| `chat.qwen.ai` | 海外 Qwen Studio，两步式 `/api/v2/chat/completions`，增量 delta | 不接入。协议和风控域名都不同 |

未登录访客大约只能试聊一轮，随后出现含 `passport.qianwen.com` iframe 的登录层。编排器把该层视为 `NEED_LOGIN`。会话 URL 形态为 `https://www.qianwen.com/chat/<32位十六进制>`。

### 1.4 页面结构

国内站是 React + CSS Modules。稳定锚点是 `role`、`aria-label`、`data-*`，哈希类名只作辅助。

**作曲器。** 输入区不是 `<textarea>`，而是 Lexical / Slate 一类的 `div[role="textbox"][contenteditable="true"]`，部分版本带 `data-slate-editor="true"`。空态时上面还有一层 `contenteditable="false"` 的占位（“向千问提问”）。直接改 `innerText` 或 `innerHTML` 不会更新编辑器内部状态，发送按钮会当空内容处理。能同步状态的页面内路径是：聚焦后对选区派发 `beforeinput`（`inputType: insertText`）或 `paste`。这两条路径产生的事件 `isTrusted === false`。方案一和方案五不走这条路径，它们用系统按键或剪贴板快捷键，让编辑器自己的监听器收到可信输入。方案二、三的 CDP `Input.insertText` / `Input.dispatchKeyEvent` 在调试器通道上是可信事件，但调试器附着本身是另一项暴露，见第 2 章。

**开关与按钮。**

| 控件 | 锚点 | 状态 |
| --- | --- | --- |
| 深度思考 | `button[aria-label="深度思考"]` | `aria-pressed="true"` 或激活类名 |
| 深度研究 | `button[aria-label="深度研究"]` | 本系统默认不打开。它会走工具调用，拉长会话并改变卡片结构 |
| 发送 | `button[aria-label="发送消息"]` | 空输入或生成中为禁用，生成中常切换成停止 |
| 停止 | 同一位置的停止语义按钮 | 出现即说明服务端已在生成 |

深度思考若未按下，旗舰模型可能不产出 `deep_think`。任务参数 `deep_think=true` 时，编排器只在 `aria-pressed` 不符时点一次开关，再读一次。读不到开关则慢轨自愈，自愈失败则 `NEED_HUMAN`，不用“盲点发送”碰运气。

**消息骨架。**

| 区域 | 锚点 |
| --- | --- |
| 滚动容器 | `.chat-content` 一类，实现时用“包含回答节点的最近滚动父元素”探测，不写死哈希 |
| 提问块 | `[data-chat-question-wrap]`、`div[data-chat-list-key$="-question"]` |
| 回答块 | `[data-chat-answers-wrap]`、`[data-msgid$="-answer"]`、外层 `[data-req-id]` |
| 思考折叠 | 类名包含 `think_content` 的节点，标题随阶段变化 |
| 正文 | `#qk-markdown-react` 或 `.tongyi-markdown` |

长会话会虚拟化列表：滚出视口的旧节点可能从 DOM 卸下。采集器只盯**本轮**回答节点，并在该节点进入视口后再读；不依赖“页面上所有历史消息一直都在”。每轮回答都从自己的节点取文本，不从 `document.body.innerText` 截取。

### 1.5 深度思考的视图状态

| 阶段 | 标题栏 | 内容区 | 采集含义 |
| --- | --- | --- | --- |
| 未开始 | 无折叠区 | 无 | 还不能声明“思考为空”，要等首个 SSE 等效 DOM 更新或超时 |
| 思考中 | “思考中...” | `think_content` 持续变长 | 只追加快照，不判定完成 |
| 正文开始 | “已深度思考（用时 X 秒）” | 思考区停止增长，正文区开始增长 | 思考文本可以封存候选版本 |
| 完成 | 标题保持“已深度思考” | 两区静默 | 与按钮态、静默窗口一起确认 |

折叠会把思考正文从可访问文本里拿掉，只留标题。只读观察器在封存前若发现内容区高度为 0 或文本只剩标题，先记录“需要展开”，由写通道点一次标题栏，展开后再读。展开用的点击也走当前方案的可信写通道，不在隔离世界里 `element.click()`。

正文是 Markdown 的 HTML 投影：围栏代码、表格、KaTeX 公式。`innerText` 对代码块通常还能保住换行和围栏文字，对 KaTeX 会得到排版后的可读文本而不是 LaTeX 源。审计与展示默认接受这个投影。任务若声明“要公式源码或工具卡片 JSON”，完成态之后走方案四的同源只读对账，DOM 文本仍作为展示兜底。

### 1.6 协议层：累积快照，而不是 OpenAI 式 delta

对话流：

`POST https://chat2.qianwen.com/api/v2/chat`

历史与详情：

- `https://chat2-api.qianwen.com/api/v2/session/page/list`
- `/api/v1/session/msg/list`

请求是 `fetch` + `ReadableStream`，不是浏览器 `EventSource`。`EventSource` 不能带 JSON 体，也带不了页面已有的签名头。公开适配器观察到的请求特征：

- `Accept` 含 `text/event-stream`
- `Content-Type: application/json`
- `x-platform: pc_tongyi`
- 页面自己的 `x-xsrf-token`、`x-deviceid`
- 阿里安全头：`clt-acs-sign`、`bx_et`、`bx-ua`、`bx-umidtoken`
- 深度思考时体字段 `deep_search` 为 `"1"`，`protocol_version` 为 `"v2"`，`biz_id` 为 `"ai_qwen"`

每一帧 `data:` JSON 是**截至该时刻的累积全文**，中间夹着 `signal/*`、`bar/*`、`paa/*` 等控制帧。谁若按 OpenAI delta 去拼接，会把同一段思考重复接上很多遍。差分必须用“新串是否以前串为前缀，否则按最长公共前缀切增量”。

深度思考开启后，主消息 `mime_type` 变为 `multi_load/iframe`：

- 思考全文在 `meta_data.multi_load[]` 里 `type === "deep_think"` 的 `content.think_content`
- 正文在主 `content`，开头嵌占位符，形如 `[(deep_think:...)]` 或 `[(multimodal_chat_think_1)]`
- 联网引用、文档卡片是同一套占位机制（`[[source_group_web_1]]`、`[(doc_common_card:...)]`），采集正文时要保留占位符旁边的可见文字，不要把整段 `multi_load` 糊进正文栏

```mermaid
flowchart LR
    subgraph proto [协议帧]
        SSE["累积 SSE 帧"]
        ML["multi_load deep_think.think_content"]
        CT["主 content 去掉占位符"]
    end
    subgraph view [视图]
        TH["折叠区 think_content"]
        MD["qk-markdown-react"]
    end
    SSE --> ML --> TH
    SSE --> CT --> MD
```

方案一、二、五只读视图。方案三可以用 CDP `Network` 听同一条响应，但监听意味着调试器域打开。方案四在第 5 章规定：要么在 `baxia.js` 已经签完名的下游做只读分流，要么等生成结束后用页面自己的 `fetch` 拉一次 `/api/v1/session/msg/list`。禁止平行再签一套 `bx-ua` 从系统外重放 `/api/v2/chat`。那样既会丢会话风控 Cookie 的时序，也会变成第二个客户端。

### 1.7 双区采集状态机

观察器维护两个缓冲区 `think_buf`、`answer_buf`，以及：

| 字段 | 含义 |
| --- | --- |
| `think_title` | `thinking` / `done` / `absent` |
| `send_btn` | `send` / `stop` / `disabled` |
| `quiet_ms` | 两区字节长度连续不变的时长 |
| `generation` | 回答节点的 `data-req-id`，防止把上一轮尾部增量算进本轮 |

封存条件（全部满足）：

1. `send_btn == send`（停止按钮已恢复为发送）
2. `think_title == done` 或任务明确关闭了深度思考且 `think_title == absent`
3. `quiet_ms >= 1200`
4. `answer_buf` 非空，或思考区已标明无法作答的可见正文

1200 ms 静默用来躲开 Markdown 二次排版和公式延迟渲染。墙钟上限默认 180 秒，超时进入 `PARTIAL` 而不是再点发送。虚拟列表若中途把节点卸掉，观察器用 `data-req-id` 重新绑定；绑定失败则保留最后一次快照并标 `DOM_DETACHED`。

HTML 到 Markdown 的转换在扩展或本机编排器里做，规则固定：

- 标题、列表、表格、链接按 DOM 结构转
- `pre code` 保留围栏和语言类名
- 图片保留 `alt` 与 `src`，不下载
- KaTeX 若节点上带原始 TeX 注解则优先注解，否则保留 `innerText` 并打标 `math_flattened`
- 不执行页面脚本，不把 HTML 送进本地模型当提示词的未转义片段

---

## 第 2 章 阿里 Web 风控与主号防封

本章只回答一件事：哪些自动化痕迹会叠进风险分，以及本系统如何让个人主号停留在“一个正常人开着日常 Chrome 慢慢问问题”的分布里。不讨论伪造 `bx-ua`，也不讨论自动拖滑块。滑块出现即熔断。

### 2.1 纵深链，从边缘到处罚

一次对话请求在浏览器里已经走完下面这条链。本系统的原则是**让页面自己走完它**，而不是在外面重造。

| 顺序 | 组件 | 它产出什么 | 本系统的态度 |
| --- | --- | --- | --- |
| 1 | 阿里云盾边缘，`acw_tc` / `acw_sc__v2` | 会话入口 Cookie，异常时直接挑战页 | 用真实浏览会话自然获得，不删不重放 |
| 2 | `awsc.js` | 按需拉起 `fireyejs`、`nc`、`uab` | 不拦截这段脚本 |
| 3 | `baxia.js` | Hook 页面的 `XMLHttpRequest` 与 `fetch`，在业务请求发出前塞入安全头；处罚时弹出 `baxia-dialog` | 不在它上游再包一层 `fetch` |
| 4 | `fireyejs.js` | 字节码虚拟机，生成约 1200+ 字符的 `bx-ua`，以及 `bx_et`（`window.etSign` 一类） | 不在 Node 里补环境去生成 |
| 5 | `um.js` | 设备标识 `bx-umidtoken`，配合 `cna` | 让同一真实 Profile 长期稳定 |
| 6 | Sufei Cookie：`tfstk`、`isg`，常与 `cna`、`ssxmod_itna` 并存 | 页面生命周期内的动态状态，长度合计常见一千多到两千字符 | 冷启动缺失会在中途被罚，所以不从零 Cookie 的脚本直打 API |
| 7 | `nc.js` 滑块 | 处罚页 `_____tmd_____/punish` 或 JSON `RGV587_ERROR`；通过后下发 `x5sec` | 人来处理。系统只负责在一秒内停手 |

`fireyejs` 的初始化还带环境自检：CSS `animationend`、`postMessage` 微任务耗时、大约 2 秒的 init 超时。纯 JSDOM 或单步调试会把内部状态打成异常，表面仍能吐出 `bx-ua` 字符串，服务端验签直接拒绝。这是“不要自建签名器”的技术原因，不是策略口号。

### 2.2 六类检测向量

**向量 1：显式自动化标志。**  
`navigator.webdriver === true`、启动参数 `--enable-automation`、ChromeDriver 残留 `$cdc_`（`/$[a-z]dc_/`）、`__webdriver_evaluate`、`domAutomation`、Playwright 的 `__pwInitScripts`。方案一、二、五使用用户已经打开的商业版 Chrome，不加这些开关。方案三即使用 Patchright，也必须 `channel="chrome"`、有头、独立持久目录，并且不去改 `navigator.webdriver` 的描述符来“伪装 undefined”。拙劣的描述符篡改本身就是信号。

**向量 2：CDP 附着的残余侧信道。**  
Chrome 136（2025-04 稳定版）起，默认用户数据目录上的 `--remote-debugging-port` 和 `--remote-debugging-pipe` 被忽略，必须搭配非默认 `--user-data-dir`。这是为了堵住木马用调试端口导出 Cookie，也顺便堵死了“给日常 Profile 加一个调试端口”的偷懒做法。  
Chrome 138 修掉了经典的 `Runtime.enable` + `Error.stack` getter 探测。Chrome 142 修掉了 `Input.dispatchMouseEvent` 里 `screenX === pageX` 的坐标特征。2026 年仍有人用原型链 `Proxy` 的 `ownKeys` 观察 `Runtime.enable` 的预览行为。结论：调试器附着没有变成“完全不可见”。方案一和方案五不打开调试器。方案二的扩展调试器只附着 Agent Window 里的那一个标签，用完释放。方案三接受“独立 Profile 上的 CDP”这一暴露，换无人值守，不拿来跑日常主号。

**向量 3：原生函数完整性。**  
`Function.prototype.toString` 应看到 `function fetch() { [native code] }`。`baxia.js` 自己已经包了 `fetch`。扩展若在 `MAIN` 世界再包一层，会出现：和霸下包装的顺序冲突、`toString` 对不上、错误栈里出现扩展的 `sourceURL`。方案四若做分流，只能包在霸下包装的下游，包装函数的 `toString` 必须仍返回原生形态，并且不能在 `window` 上挂自定义名。默认路径连这个包装都不做，生成结束后用一次同源只读请求对账，调用栈就是页面自己的。

**向量 4：事件可信度与动作生物特征。**  
`event.isTrusted` 是只读的。`dispatchEvent(new MouseEvent(...))` 恒为 `false`。滑块和部分控件会直接丢弃。UIA 的 `InvokePattern` 走无障碍通道，页面侧同样不可信。`SendInput` 进入系统输入队列后，DOM 事件为可信；网页渲染进程读不到内核的 `LLMHF_INJECTED`。只有本机安全软件的低级钩子才看得见注入标志。USB HID 在固件层是标准键鼠，内核注入标志为 0。  
轨迹上，阿里滑块采样从按下到移动到抬起：点数、间隔、先快后慢、Y 轴不能全程为 0、35–90 ms 的按压驻留。本系统的写通道按第 7 章的 Minimum-Jerk 发生器产生移动，点击驻留落在这个区间。日常提问不主动去找滑块练手。

**向量 5：图形与环境完备性。**  
Canvas、WebGL 渲染器、音频、字体、`screen` 与 `outerWidth` 的一致性、`hardwareConcurrency`、`deviceMemory`。无 GPU 容器里的 SwiftShader、无头壳、`Chrome for Testing` 的 UA 和缺失的专有编解码器，都会抬分。本系统使用本人正在用的商业版 Chrome 和真实核显，不注入一份“理想指纹”。指纹与真实机器不一致，比指纹普通更危险。

**向量 6：服务端时序。**  
环境全真也挡不住节奏。公开适配观察：大约连续十轮背靠背的长流式对话、或只带登录 token 却没有 `cna` / `tfstk` / `isg` 的冷请求，会从 SSE 变成处罚 HTML 或 `RGV587`。编排器因此是单飞，两轮之间有阅读停顿，并且只在已经打开过千问页面的热会话上工作。

### 2.3 主号十条铁律

1. 只使用本人日常 Chrome Profile 或其上长期养过的专用 Profile。不使用无头壳，不使用 Chrome for Testing 作为主号载体。
2. 不伪造硬件指纹、不改 `webdriver` 描述符、不随机化 Canvas。
3. 不加 `--enable-automation`，不在默认用户数据目录上尝试调试端口。
4. 主号路径不长期附着 `chrome.debugger`。若某方案必须附着，附着范围是单个受控标签，任务结束立即 `detach`。
5. 不在 `MAIN` 世界替换 `fetch` / `XMLHttpRequest`，除非走方案四并且包装位于 `baxia.js` 下游、`toString` 保持原生形态、任务结束即卸掉。
6. 写入页面的点击和按键必须是可信事件。隔离世界的 `element.click()` 只允许用于“读失败后的自检”，不能作为发送动作。
7. 全局单并发。两轮提问之间插入阅读冷却，间隔取对数正态，中位 25 秒，下限 8 秒，并加上与回复长度成正比的附加停顿（每 400 字加 1 秒，上限再加 40 秒）。
8. 输入混合使用短句逐字与偶尔粘贴，粘贴前有 200–600 ms 的聚焦停顿；不在 50 ms 内灌入两千字。
9. 观察到 `baxia-dialog`、处罚页、`RGV587`、连续两次空流，1 秒内停止输入，状态进入 `NEED_HUMAN`，当天不再自动重试该账号。
10. 不并行跑两套执行面打同一个会话，不在生成未完成时再次发送，不在 `SUBMIT_UNKNOWN` 未对账时重启一轮。

### 2.4 暴露面预算

把“主号可接受”定义成下面这张预算，第 5 章的每套方案都要回填它。

| 预算项 | 主号允许 | 不允许 |
| --- | --- | --- |
| 浏览器二进制 | 已安装的商业版 Chrome | 无头壳、CfT、改过的 Chromium |
| 调试器 | 无，或单标签短时附着 | 常驻黄条、`Runtime.enable` 挂满所有标签 |
| 主世界原型 | 不修改 | 持久 Hook `fetch` |
| 输入 | 可信事件 + 非直线轨迹 | `isTrusted=false` 的发送点击 |
| 身份 | 原有 `cna` / `umidtoken` 连续 | 每次启动新指纹 |
| 节奏 | 单飞 + 阅读冷却 | 背靠背十连 |
| 处罚 | 立即人工 | 自动拖滑块、自动重放被罚请求 |

```mermaid
flowchart TB
    req["业务请求"] --> baxia["baxia.js 签名"]
    baxia --> fy["fireyejs 产出 bx-ua"]
    fy --> edge["云盾 + 服务端风险分"]
    edge --> ok["SSE 累积帧"]
    edge --> bad["punish 或 RGV587"]
    bad --> stop["1 秒内熔断 NEED_HUMAN"]
```

第 0–2 章到此锁定需求、页面契约和风控预算。后面的技术和方案都回指这三章，不再放宽铁律。

## 第 3 章 技术全景与选型

本章分三段：点名的 8 个产品各自真实在做什么、扩大范围后的协议 / 驱动 / 操作系统三条线、以及哪几件东西能进本系统的底座。版本以 2026-11 公开仓库与文档为准。评价标准只有第 2 章的暴露面预算和第 0 章的 CPU 预算，不按 Star 数排名。

### 3.1 八个参照系

#### 3.1.1 OpenClaw

OpenClaw（`openclaw.ai`，仓库 `openclaw/openclaw`）是本地 Gateway 型助手，2026-11 稳定版在 `v2026.11.1`–`v2026.11.3`，MIT，TypeScript，Node 24+。浏览器层是 CDP + Playwright `connectOverCDP` + MV3 扩展中继。

和本系统相关的是四条 Profile，而不是它的聊天网关：

| Profile | 驱动 | 登录态 | 对本系统的含义 |
| --- | --- | --- | --- |
| `openclaw` | Gateway 拉起独立 Chrome，端口段 18800–18899，故意避开 9222 | 独立 `userDataDir` | 等价于方案三的“另起炉灶”，主号不用 |
| `chrome` | MV3 扩展连 `ExtensionRelayBridge` | 日常 Chrome，只共享名为 OpenClaw 的标签组 | 扩展中继的成熟参考。桥在服务端合成 `Browser.*` / `Target.*`，把主页面和子 frame 复用到一条 `chrome.debugger` 上 |
| `user` | Chrome DevTools MCP `--autoConnect` | 日常 Chrome | 会碰到 Chrome 144+ 的 “Allow remote debugging?” |
| `remote` | 远程 CDP | 云浏览器 | 主号 IP 和指纹都换了，排除 |

2026 年的 Browser Relay Auth v2 用 nonce 绑定的双向 HMAC，不把长期密钥放进握手明文。快照是 Playwright AI 快照 / 无障碍树引用（`f1e12`、`e12`），动作强制用 ref，不用 CSS 选择器。`navigate` 会内联返回紧凑可交互快照。

可借鉴：标签组隔离、ref 而不是选择器、扩展中继不要远程调试弹窗。  
不整体搬入：Gateway、多通道、Playwright 会话对一个只访问千问的状态机过重，而且 `chrome.debugger` 与第 2 章主号预算冲突。扩展模式仍有调试横幅。

#### 3.1.2 WorkBuddy

`workbuddy.ai` 是腾讯的闭源桌面办公 Agent（不要和 `work-buddy.ai` 搞混）。2026 年桌面端约 `v5.2.5+`，配套 Chrome 扩展约 `v0.2.8`。结构是 Electron 外壳 + CodeBuddy CLI（`@genie/agent-cli`）+ Native Messaging 宿主 `com.workbuddy.extension`。扩展 ID `ajnnogdfpilbhkeggdjlcokgglijmdde`。宿主把浏览器包成名为 `browser-mcp-*` 的本地 MCP，对上暴露约 32 个固定工具。

关键设计是这 32 个工具里**没有** `Runtime.evaluate`。主路径是截图加坐标，备选是扩展侧 DOM 定位。标签组叫 WorkBuddy，后台跑，不抢用户当前窗口。

可借鉴：Native Messaging 比本地 WebSocket 少一个端口；把“任意脚本”从模型工具表里删掉；标签组收尾时关掉。  
不搬入：闭源、绑定腾讯客户端；整屏截图在纯 CPU 上不能当每步感知（见 3.3 视觉路线）。坐标点击若走 `Input.dispatchMouseEvent`，可信度没问题，调试器附着仍在。

#### 3.1.3 Claude Computer Use

2026 年正式工具集是 `computer_toolset_20260801`。模型返回 `toolset_name: "computer"` 下的一批成员工具，而不是早期那种单个 `computer` 工具加 `action` 字段。成员包括 `screenshot`、`zoom`、点击族、拖拽、滚动、`type`、`key`、`hold_key`、`wait` 等。`zoom` 传入全屏坐标矩形取局部高分辨率，之后的点击仍用全屏坐标系。配套还有网页向的 `browser` 工具集和 Claude in Chrome 扩展。官方参考实现是容器里的 Xvfb + Mutter + xdotool，模型只看见像素。

坐标缩放用 `scale = min(1, 1568 / max(w,h), sqrt(1_150_000 / (w*h)))`。工具集声明本身大约固定占用 4500 输入 token 量级，每张缩放截图大约再加 1000–1800。纯 CPU 上若还要本地视觉模型理解截图，一步就是几十秒，而且 4B 文本模型根本不吃图像。

可借鉴：输入发生在操作系统层，浏览器里零 CDP。这是方案五的思想来源。  
不搬入：把感知也做成像素循环。千问 DOM 是可读的，用截图找“发送”按钮是在纯 CPU 上最贵的办法。Claude in Chrome 是订阅产品，不能当本地底座。

#### 3.1.4 CoWork-OS

`CoWork-OS/CoWork-OS`，MIT，2026-11 约 `v0.5.62`。Browser V2 默认后端是 Electron `WebContentsView`，主进程 `webContents.debugger` 发 CDP。另外有 `playwright-local`、`browser-use-cloud`、`camofox`。`external-cdp` 被明确拒绝：不附着用户已经打开的 Chrome，因为那些已有连接进不了任务的网络策略。

快照节点带短生命周期 `ref`、role、name、bounds。页面一变，旧 ref 立即作废。Profile 默认按工作区隔离；`profile: "user"` 要在系统 Chrome 没占用该目录时才能启动。macOS 上可以只读导入 Cookie，Windows 上没有同等钥匙串路径。

可借鉴：ref 的强时效，和本系统“虚拟列表卸节点后必须重新绑定 `data-req-id`”是同一类问题。Camofox 说明 Firefox 内核级指纹是另一条产品线。  
不搬入：它拒绝附着正在运行的日常 Chrome，和“用户已经登录千问”直接相反。内嵌 Electron 的 TLS 与字体栈也不是用户的日常 Chrome。

#### 3.1.5 browser-harness

`browser-use/browser-harness`，2026 年 README 写明它已经成为 Browser Use 的引擎，独立版本约 `0.1.13`。它自己没有 LLM 循环。链路是：外部 Agent 跑一段 Python → Unix socket 或 Windows 上 TCP 回环加令牌 → 常驻 daemon → CDP。核心依赖很薄：`cdp-use`、`fetch-use`、`pillow`、`websockets`。

省 token 的做法是 `Accessibility.getFullAXTree` 在 Python 里滤完再打印给模型，再用 `DOM.getBoxModel` 算中心点，`Input.dispatchMouseEvent` 点击。`fill_input` 对受控输入发真实键并补 `input`/`change`。`agent_helpers.py` 放在工作区，允许编码 Agent 把缺的动作成函数沉淀；`domain-skills/` 是站点手册。

2026-10-14 的 BU Bench V2.1（100 个公开任务，Codex CLI 0.144.6 + GPT-6 Luna）中位输入约 **2.1M token/任务**，中位耗时 9.0 分钟，中位成本约 $0.042。对照：agent-browser 0.39.1 为 3.3M / 11.2 分钟，Playwright MCP 0.0.83 为 4.4M / 8.3 分钟，Chrome DevTools MCP 1.10.1 为 6.7M / 8.7 分钟。这个数字证明“先在本地过滤再给模型”有效。它不能证明本系统应该上 Python REPL：本系统的模型调用次数设计成接近于零，token 账本的大头在审计那 600 token，不在浏览器循环。

登录态走 `chrome://inspect/#remote-debugging`，Chrome 144+ 首次要人点 Allow。文档写明 Chrome 136+ 在默认目录上禁用调试端口。

可借鉴：站点微技能手册（本系统的千问技能就是一份 domain skill）、本地预过滤、守护进程把 CDP 连接保持住。  
不作为主号底座：它就是一条 CDP。`BH_TAB_MARKER` 还会改 `document.title`，千问页面脚本若扫标题会看见，主号路径应关掉这类标记。

#### 3.1.6 browser-use 1.0

`browser-use==1.0.2` 把 0.x 的“每步高亮 DOM → JSON 动作”删掉了。1.0 的句子是：模型、一个 REPL、CDP。Pydantic AI 驱动模型在子进程 Python REPL 里写代码，调用 browser-harness 0.1.13。`mode='ultrafast'` 用 `browser_use/ax.py` 把无障碍节点收成 `bu.click(41)` 这种短指令。默认 `Browser()` 开的是云端隐身浏览器；`Browser('chrome')` 才附着本机 Chrome。Windows 上 `Browser('chrome', profile='Work')` 要先关掉原 Chrome，因为 Cookie 文件锁。

REPL worker 以当前 OS 用户跑模型写出的 Python。一个只为了点“发送”的系统不需要这个执行面，它还多一个提示注入后任意代码的问题。可借鉴的是 `ultrafast` 的紧凑引用，以及“一个单元里做完多步”对自愈慢轨的启发：慢轨一次调用应该返回“点击哪个框、期望看到什么文本”，而不是再进入十步循环。

#### 3.1.7 agent-browser

`vercel-labs/agent-browser`，Apache-2.0，2026-10/11 约 `v0.39.x+`。CLI 和守护进程都是 Rust。默认浏览器是 Chrome for Testing。`--auto-connect` 读 `DevToolsActivePort` 或探测 9222/9229，`--pin-tab` 把标签锁住，误关返回 `tab_gone`。`--profile` 把系统 Profile 拷到临时目录只读启动，Windows 上原 Chrome 开着会撞文件锁。快照命令 `snapshot -i -c -d` 输出 `- link "..." [ref=e1]`，自动内联 iframe。引擎选项还有 Lightpanda、实验性 Obscura（`--stealth`），以及 Browserbase 等云厂商。

可借鉴：`--pin-tab` 的互斥思想（本系统用文件锁保证单飞）、快照裁剪标志、Rust daemon 的启动开销确实低。  
不作为主号底座：默认 CfT 带测试版 UA 和缺失的专有编解码器，正是第 2 章向量 5 要避开的。`--auto-connect` 仍是 CDP。Obscura 在 2026 年仍是实验引擎，不能把主号交给它。

#### 3.1.8 BrowserSkill

`Tencent/BrowserSkill`，MIT，2026-10/11 的 `v0.4.0` / `v0.4.1`，协议 1.7 / 1.8。Rust 的 `bsk` 同时是 CLI 和 `bsk daemon`，扩展是 WXT 写的 MV3（要求 Chromium 125+，扁平 OOPIF）。链路：

`bsk` → 命名管道或 `~/.bsk/run/daemon.sock` → daemon → `ws://127.0.0.1:52800` → 扩展 Service Worker → `chrome.debugger` + WebExtensions API。

默认在已登录 Profile 里开一个独立 **Agent Window**，`--no-focus`，不碰用户当前标签。需要用户已有标签时走 `bsk tab borrow`，结束归还；用户把标签拖出窗口则会话终止。`bsk cooperate sync` 签发绑定会话、标签、文档版本和人类干预纪元的 `context_id`。动作可带最多 16 条 `--expect`（文本、value、checked、属性、存在性）。人一旦动手或文档变了，旧凭证作废。观察走 `packages/vom` 的语义树，`@eN` 引用，`--max-tokens` 和游标分页。Canvas 节点标记为需要截图，截图 id 两分钟过期，点击必须带回 `capture_id` 和图像坐标。

这是方案二的底座，原因不是“腾讯出品”，而是它已经实现了本系统需要的四件事：真实 Profile、独立窗口不抢焦点、人类干预自动作废、动作前断言。协议从 1.3 到 1.8 有破坏性升级，锁定版本是使用条件。调试器暴露仍在，所以它不是主号安全天花板，方案一才是。

#### 3.1.9 八个产品对照

| 产品 | 感知 | 输入可信度 | 日常 Chrome 登录态 | 调试器 | 纯 CPU 上的步代价 | 主号角色 |
| --- | --- | --- | --- | --- | --- | --- |
| OpenClaw `chrome` | 无障碍 ref | CDP 可信 | 标签组 | 有 | 低 | 中继参考 |
| WorkBuddy | 截图为主 | CDP 可信 | 标签组 | 有 | 高（图像） | Native Messaging 参考 |
| Claude Computer Use | 纯像素 | OS 级可信 | 默认不复用 | 无 | 极高 | 方案五的输入思想 |
| CoWork-OS | ref + bounds | CDP 可信 | 拒绝附着运行中的实例 | 内嵌有 | 低 | 反面教材：隔离优先 |
| browser-harness | Python 滤 AX 树 | CDP 可信 | inspect 授权 | 有 | 低 | 技能手册思想 |
| browser-use 1.0 | ultrafast AX | CDP 可信 | `Browser('chrome')` | 有 | 低，但 REPL 重 | 不引入 |
| agent-browser | `snapshot -i -c` | CDP 可信 | `--auto-connect` 或拷贝 Profile | 有 | CLI 极低 | 单飞锁参考 |
| BrowserSkill | VOM `@eN` | CDP 可信 | Agent Window | 有，可限单窗 | 低 | 方案二底座 |

### 3.2 扩大范围后的三条技术线

#### 3.2.1 扩展与新协议

**BrowserMCP（`@browsermcp/mcp`）。** 每个 AI 应用起一个瘦进程，每用户一个 daemon。扩展经 `ws://127.0.0.1` 的 `/extension` 连上，端口在 9010、9011、9012 间选取。配对是 ECDSA P-256：hello、welcome（承诺）、pairingNonce、pairingReveal，双方各自出随机数，六位数对得上才 Allow。daemon 用 Playwright `connectOverCDP` 连自己桥出来的 `/cdp`。扩展只对 Agent 正在用的标签 `chrome.debugger.attach`。隐藏标签在 Chrome 里不绘制，所以它在输入和截图前检查 `document.visibilityState`，并用 `Emulation.setFocusEmulationEnabled`。这点对千问很重要：后台标签的截图会是空白，合成输入可能挂起后补发。方案二若在后台窗口操作，优先 DOM 观察，不依赖截图。BrowserMCP 的单工具 `browser` 和配对流程值得抄进自研扩展的威胁模型，但它仍是调试器中继，替代不了方案一。

**Stagehand V3 `ActCache`。** `browserbase/stagehand` 的 `packages/core/lib/v3/cache/ActCache.ts`：`act("点发送")` 第一次走模型，成功后把选择器序列写入 `cacheKey.json`。下次 `tryReplay` 先 `waitForCachedSelector`（默认结算超时 15 秒），再确定性执行。回放失败且 `selfHeal` 开启时，`actHandler` 重新抓混合快照，让模型给新选择器，成功则 `refreshCacheEntry`。`model: "auto"` 没有本地客户端，不走本地回放。2026-01 的 PR 1498 修了一个真 bug：自愈重推理要用用户指定的模型，不能悄悄换成默认模型。本系统的微技能缓存按这个语义做，但执行器换成第 7 章的状态机，自愈模型换成本地 4B，而且缓存键包含千问前端构建号（从页面脚本 URL 的哈希读），构建一变就视为未命中，避免把旧选择器在新页面上点错位置。

**Chrome WebMCP（`navigator.modelContext`）。** Chrome 149/150+ 的 Origin Trial / 特性开关。网站自己 `registerTool`，Agent 调结构化工具，宣称可少消耗大量 DOM token。千问没有注册这些工具。本系统不把路线押在站点自愿合作上。

**chrome-devtools-mcp v1.0。** 2026-05 稳定版，20 多个工具（`navigate_page`、`click`、`fill`、`take_snapshot`、`evaluate_script` 等），配合 Chrome 144+ `--autoConnect`。官方目标是可观测性，不做隐身。它会开调试域。只适合方案三这类已经接受 CDP 的独立 Profile 做调试，不进主号热路径。

**Tampermonkey `USER_SCRIPT` 世界。** Chrome 138 起，“允许用户脚本”是扩展详情页上的单独开关，不再是全局开发者模式。脚本默认在 `userScripts.configureWorld()` 的隔离世界，`unsafeWindow` 才能碰到页面函数。事件仍然 `isTrusted === false`。适合方案四的快速试验，不适合作为发送通道。Chrome 152 曾有 `userScripts` 静默失效、要重开扩展的问题，主路径不依赖它。

#### 3.2.2 五个反检测驱动

| 驱动 | 它改了哪一层 | 对千问主号的判断 |
| --- | --- | --- |
| Patchright | 改 Playwright 驱动：去掉每帧 `Runtime.enable`，选择器走 `Page.createIsolatedWorld`，关掉 Console 域，去掉一批自动化开关，可穿透 closed shadow | **方案三的驱动**。条件：有头、`channel="chrome"`、持久非默认目录、不伪造指纹。代价：`page.on("console")` 不可用；隔离世界读不到主世界变量。DOM 采集放隔离世界是优点，和“不要污染主世界”一致 |
| Rebrowser-patches | Puppeteer/Playwright 补丁。`alwaysIsolated` / `addBinding` / `enableDisable` 三种拿上下文的方式 | `enableDisable` 在开关瞬间仍有窗口；`addBinding` 会在全局留名字。主号不用。方案三已选 Patchright，不再叠第二套补丁 |
| DrissionPage | Python 直连 CDP，默认不开 `Runtime.enable` | 元素操作会在主世界密集调用 `scrollIntoViewIfNeeded`、`getComputedStyle`、`offsetParent`。页面若预先包装这些函数做计数，就能认出这条调用链。千问站上不采用 |
| Nodriver | Python 异步直连，全程不发 `Runtime.enable` | 脚本跑在主世界，没有隔离世界。同样受 Chrome 136 默认目录限制。不作为主号驱动 |
| Camoufox | Firefox C++ 层改指纹，Juggler 协议，`page.evaluate` 在沙箱 | 它是 Gecko 不是 V8。国内 C 端流量里 Firefox 占比低，UA 分布本身会扎眼。Juggler 在重 SPA 上有过管道假死。千问不用 Firefox 内核 |

方案三只保留 Patchright 这一条，其它四个写入排除表，避免实现时“再加一层更隐蔽”把指纹改得失真。

#### 3.2.3 操作系统与外设

**Windows UI Automation。** 读无障碍树不需要调试端口，页面 JS 不知道有人在读。Chromium 的树是懒构建的，第一次查询大页面要几百毫秒到一秒以上。`Invoke` / `SetValue` 到达页面时 `isTrusted` 为 false，不能用来点发送，也不能用来拖滑块。断开的远程桌面会话里，隐藏渲染器可能不再更新树。方案五用 UIA 做几何锚定和文本兜底，动作用 `SendInput`。

**Win32 `SendInput`。** 事件进当前桌面的输入队列，DOM 侧可信，带硬件时间戳。渲染进程不能自己装 `WH_MOUSE_LL` 去读 `LLMHF_INJECTED`。约束是：目标窗口必须在前台，锁屏、最小化、无交互桌面会把输入丢掉；同一时刻只有一个前台，不能两路并行。坐标来自扩展报告的屏幕包围盒或 UIA bounds，再加一次视口滚动校正。

**ESP32-S3 / RP2040 USB HID。** 开发板枚举成标准 USB 键盘和鼠标，主机内核里的注入标志为 0。方案一的写通道把“轨迹点列”发给本机小服务，小服务要么调用 `SendInput`，要么交给串口另一头的固件去回报 HID。编排器不关心是哪一种，契约相同：绝对坐标移动、按下、驻留、抬起、Unicode 文本或剪贴板快捷键。固件不实现滑块求解。这块硬件是写通道的可替换后端，大约一块 S3 开发板的成本，用来在确认本机安全软件会看 `LLMHF_INJECTED` 时再升级，而不是第一天就焊电路。

**视觉三条，全部不进主循环。**

| 路线 | 它是什么 | 纯 CPU 上为什么不进热路径 |
| --- | --- | --- |
| OmniParser V2 | 微软 2025-02 的截图解析器，把像素收成可点击框和图标说明。OmniParser+GPT-4o 在 ScreenSpot Pro 上约 39.6。OmniTool 是带 Docker 的 Windows 沙箱 | 检测模型本身要跑一遍视觉推理。16 GB 机器上再驻留一个视觉模型和 4B 文本模型会撞穿 4.8 GB。千问控件有名字，不需要从像素里找 |
| Midscene.js | 字节 web-infra 的视觉 E2E，接 Playwright / Puppeteer / 桌面 Chrome，也接 UI-TARS 与 Qwen-VL。跨 Web、ADB、WDA、HDC、桌面 | 每步一张图。和 Claude Computer Use 同一类代价。可在方案五“UIA 也读不到”时做一次离线诊断，不进每问一次的循环 |
| UI-TARS | 字节的端到端 GUI 模型。公开权重里 `UI-TARS-1.5-7B` 是能下载的量级；UI-TARS-2 技术报告（2025-09）给出 Online-Mind2Web 88.2、OSWorld 47.5，旗舰权重不是随手可下的桌面模型 | 7B 视觉模型的权重大于本机文本评审预算。OSWorld 不到五成说明长程点击不能当主号的确定性通道。误点发送比慢更糟 |

慢轨自愈的输入是扩展或 UIA 给出的**文本树**（role、name、可见文本、包围盒），不是截图。本地 4B 没有视觉编码器要加载。这是有意选的，用来守住 4.8 GB。

### 3.3 底座裁决

进入系统的组件：

| 层 | 选择 | 不选 |
| --- | --- | --- |
| 编排 | 自研持久化状态机 + 千问微技能缓存（ActCache 语义） | browser-use REPL、OpenClaw Gateway、CoWork 整包 |
| 主号的读 | 无 `debugger` 权限的 MV3，仅 `ISOLATED` + Native Messaging | 主世界 Hook、油猴当主路径 |
| 主号的写 | `SendInput` 轨迹服务，接口预留 USB HID | `dispatchEvent`、UIA Invoke |
| 后台免打扰 | BrowserSkill `v0.4.x` 锁协议 1.8，外面包同一套状态机 | 同时再装 OpenClaw 扩展打同一页面 |
| 无人值守独立号 | Patchright + 有头商业版 Chrome + 独立目录 | Camoufox、DrissionPage、CfT、云端隐身浏览器 |
| 协议原文 | 完成后一次同源 `msg/list`；分流器仅作显式升级 | 系统外重放 `/api/v2/chat` |
| 评审 | `llama-server` + `Qwen3.5-4B` | 云端裁判、开思考模式的小模型 |
| 视觉 | 不驻留 | OmniParser / UI-TARS / 每步截图 |

```mermaid
flowchart TB
    skill["千问微技能缓存"] -->|命中| sm["状态机"]
    skill -->|未命中或矛盾| slow["本地 4B 文本自愈"]
    slow --> skill
    sm --> read["读通道: ISOLATED 或 VOM 或 UIA"]
    sm --> write["写通道: SendInput 或 HID 或受控 CDP"]
    read --> qwen["qianwen.com"]
    write --> qwen
    qwen --> read
    sm --> judge["llama-server 审计"]
```

第 5 章的五套方案都是这张图的执行面插口，上面的编排和审计不动。

## 第 4 章 纯 CPU 上的本地模型与弱评强审计

### 4.1 内存账本

16 GB 是贴在机器上的数字，不是推理能用的数字。Windows 11 本身、浏览器（千问页加上扩展）、编排进程和文件缓存会先拿走一块。本设计把“编排 + 推理”的安全可用物理内存按 **6–8 GB** 规划，把推理进程工作集硬顶定在 **4.8 GB**。顶破之后 `pagefile.sys` 开始按页换入权重，解码会从每秒十几 token 掉到不到 1 token，整机跟着卡。

占用拆开是：

`工作集 ≈ GGUF 权重 + KV Cache + 计算缓冲（约 0.25–0.45 GB）+ llama-server 自身（约 50–100 MB）`

KV 随上下文变长。千问深度思考经常是两三千字中文，加上正文和固定系统提示，审计输入按 3K–8K token 设计，上下文窗口开 **8192**，槽位数 `--parallel` / `-np` 为 **1**。开两个槽会把 KV 翻倍，16 GB 机器上没有这个必要，审计和自愈不会同时跑（自愈发生在采集完成之前，审计在采集之后）。

### 4.2 两条 CPU 性能陷阱

解码速度大致是“有效内存带宽 / 每个 token 要读的权重大小”。

**单通道。** 不少 16 GB 笔记本是一根 16 GB 内存条。DDR5-5600 单通道理论带宽 44.8 GB/s，实测流式读大约能到 30 GB/s 出头；单通道 DDR4-3200 往往只有约 18–21 GB/s。双通道 DDR4-3200 大约 36–40 GB/s，双通道 DDR5-5600/6000 大约 62–72 GB/s。后文的 15–24 token/s、17–29 token/s 是双通道、6–8 个性能核上的区间。单通道机器把解码速度乘 **0.55–0.65**，审计墙钟从 12–25 秒变成大约 20–40 秒。这仍然可以接受，因为审计不在提问的关键路径上堵千问。不要为了追速度去上更小的量化到 `IQ2`，中文审计会先坏。

**大小核。** Intel 12 代及以后的 P 核 + E 核，Linux 上的 llama.cpp 会避开 E 核，Windows 构建默认把逻辑核都算进去。矩阵乘在每步屏障处等最慢的线程。公开的 Win11 实测里，同样的 i7，默认线程数跑在混合核上，速度可以只有“只钉 P 核”的一半；进程若被整个塞进 E 核，会掉到每秒个位数以下。启动参数 `-t` 设成 P 核数量（常见 6 或 8），不要设成任务管理器里的逻辑处理器总数。`-np 1` 保持不变。

核显不是基线，是可选项。Intel Core Ultra 的 Arc 核显或 12/13 代 Iris Xe 能通过 OpenVINO 共享内存跑 INT4 4B，解码有机会到大约 30 token/s，预填充更快。前提是只加载约 2.5 GB 的 4B，并压缩 KV。核显和 Chrome 抢的是同一 16 GB，8B 模型禁止放到核显上。检测不到 OpenVINO GPU 设备时，编排器无声退回 llama.cpp CPU，不把核显写成依赖。

### 4.3 模型

2026-03-02 开源的 Qwen3.5 小模型有 0.8B、2B、4B、9B。`Qwen3.5-4B` 文本塔约 4.2B 参数：32 层里 24 层是 Gated DeltaNet（循环状态，长度增加几乎不加 KV），8 层是 GQA 全注意力（16 个 Q 头 / 4 个 KV 头，head 维 256）。词表 248,320，和嵌入绑定。原生上下文 256K。纯文本推理不要加载 `mmproj`。默认 `enable_thinking=false`。一旦打开思考，这个尺寸会把输出拉得很长，纯 CPU 上一次审计会从半分钟变成许多分钟，而且和 llama-server 的 JSON 语法约束有冲突（见 4.4）。

`Qwen3-4B-Instruct-2507`（2025-08 这一支）是 36 层稠密 GQA，词表约 152K，上下文 256K，没有混合注意力，但指令遵循稳，KV 会随长度线性涨。8K 窗口内两者内存差不多，思考链经常超过 8K 时 3.5-4B 更不容易撞窗。

| 模型 | 量化文件量级 | 8K、q8 KV 下的进程内存 | 解码 token/s（双通道） | 裁决 |
| --- | --- | --- | --- | --- |
| `Qwen3.5-4B` | `Q4_K_M` 约 2.5–2.8 GB | 3.1–3.4 GB | 15–24 | **默认**。长思考链 KV 更省 |
| `Qwen3-4B-Instruct-2507` | `Q4_K_M` 2.50 GB；`Q5_K_M` 2.89 GB | 约 3.3 / 3.7 GB | 17–29 | 平替。清单遵循若实测更稳就换它，工作集仍低于 4.8 GB |
| `Gemma 3 4B` QAT `Q4_0` | 约 2.35 GB | 2.9–3.3 GB | 18–26 | 速度好，中文弱于 Qwen，不作为千问审计默认 |
| `Phi-4-mini-instruct` | `Q4_K_M` 约 2.49 GB | 3.2–3.5 GB | 11–23 | 中文和千问语体都不优先 |
| `CompassJudger-2-7B-Instruct` | `Q4_K_M` 约 4.68 GB；`IQ4_XS` 约 4.25 GB | **5.1–5.5 GB** | 9–18 | 上海人工智能实验室基于 Qwen2.5-7B 的评审模型，JudgerBenchV2 等均分能靠近 200B 级通用模型。**工作集超过 4.8 GB 硬顶**。仅当空闲物理内存连续 30 秒 ≥ 7.5 GB，且配置显式 `judge_profile=compass` 时，编排器才允许加载，并在加载前把 Chrome 的多余标签算进水位 |
| `Qwen3-8B` / `Qwen3.5-9B` / Ministral 3 8B | Q4 文件 4.8–5.8 GB | 5.6–6.7 GB | 7–18 | 6 GB 空闲档会换页。不进默认档 |
| `Qwen3-30B-A3B` | `Q4_K_M` 约 18.6 GB，即便 Q2 仍要 10 GB 以上 | 远超物理内存 | 在 16 GB 上实测会掉到约 1 token/s 量级 | **淘汰**。MoE 只减少激活计算，不减少必须驻留的专家权重 |

`CompassJudger` 的 32K 上下文对单次审计够用。它和 4B 共用同一套 JSON 模式和系统提示，避免两套口径。

打开 `enable_thinking=true` 的 4B/9B 不采用。Qwen3.5 小模型在思考模式下输出量会膨胀到非思考模式的数倍；CPU 时间线性变差，而且第 4.5 节要把“先摘录再裁决”放进 JSON 字段，不需要模型自己再生成一段 `<think>`。

### 4.4 运行时

| 运行时 | 额外内存 | 前缀缓存 | 结构化输出 | 本机角色 |
| --- | --- | --- | --- | --- |
| `llama-server` 官方 `win-cpu-x64` | 约 50–100 MB | 槽位最长公共前缀、`--cache-reuse`、`--slot-save-path` 落盘 | GBNF 与 OpenAI 式 `json_schema` | **唯一默认** |
| `ik_llama.cpp` | 同样低 | 慢于上游新特性 | 有，但 PEG 解析器滞后 | 仅当预填充成为实测瓶颈且模型是普通稠密结构时再评估。Qwen3.5 的 DeltaNet 在分支上的支持落后于上游，默认不押 |
| Ollama | 守护进程多占约 300–500 MB | 会话内隐式缓存，OpenAI 兼容接口锁不住槽位，也不能落盘 KV | `format` 可传 JSON Schema | 开发时临时替换可以，生产路径不用。多出来的内存直接吃掉 4.8 GB 的余量 |
| OpenVINO GenAI | 约 200–350 MB，外加 `.ovcache` | 前缀缓存默认开，KV 可压到 u8/u4 | XGrammar 一类的 schema 掩码 | Intel 核显加速的可选后端，只承载 4B |
| ONNX Runtime GenAI / Foundry Local | 同尺寸模型进程往往比 GGUF 大三成以上 | 非 CUDA 路径上重复请求仍可能重算提示 | 预览 | 不选 |

`llama-server` 目录里要保留全部 `ggml-cpu-*.dll`。Intel 12 代及以后应加载 `ggml-cpu-alderlake.dll`（AVX2 + VNNI），AMD Zen 4 加载 `ggml-cpu-zen4.dll`。日志里看不到对应 DLL，就还在用通用核，速度会虚低。

2026 年的 `llama.cpp` 问题 20345 / 23775：模型模板的思考开关和 `response_format.json_schema` 同时开启时，语法约束会失效，推理文本漏进另一个通道，或者 PEG 解析器直接 500。本系统的启动参数因此写成：

- `--jinja`
- 聊天模板参数里 `enable_thinking` 为 false
- 上下文 8192，槽位 1，KV 量化 `q8_0`
- `--cache-reuse 256`，系统提示放在固定前缀上
- 线程数等于 P 核数
- 需要从原文摘句时加 `--spec-type ngram-cache`。没有草稿模型，靠提示里已经出现过的 n-gram 猜下一个 token。摘录字段和原文重合时，解码可以明显高于上表的裸数字；自由发挥的批判句则几乎没有加速。不要为此再加载一个草稿模型。

自愈调用和审计调用共用一个槽。系统提示不同，前缀缓存会在切换时失效。这是故意的：两种调用每天各几次，不值得为它们开双槽。顺序永远是“自愈全部结束 → 卸载自愈上下文 → 再发审计”。不要并行。

水位看门：每 5 秒读一次进程工作集和系统可用物理内存。工作集超过 4.8 GB，或可用物理内存低于 1.2 GB，立即拒绝新的生成，正在生成的审计允许跑完当前 600 token 上限，然后卸载。`judge_profile=compass` 在加载前若可用内存低于 7.5 GB，自动降回 `Qwen3.5-4B` 并在 UI 上标明降级，不硬加载 7B。

### 4.5 评价到底在评什么

旗舰模型的参数量比本地模型高两个数量级。让 4B 凭记忆判断“这个 API 参数是否存在”“这道题的数是否算对”，会系统性犯错：真阳性（认出好答案）可以很高，真阴性（抓住坏答案）在多项 LLM-as-judge 研究里经常低于三成；被评模型越强，评审越宽容。冷门事实被 4B 误判成幻觉，是这个项目里比“漏掉一个真实错误”更伤可信度的失败，因为 UI 会把错误指控展示给同一个人。

所以审计员的角色是闭卷文本核对，不是第二意见领袖。它只使用三份文本：用户问题、千问思考、千问正文。它不联网，不调用千问。

```mermaid
flowchart LR
    raw["问题 + 思考 + 正文"] --> py0["Python 预检"]
    py0 --> llm["4B JSON 审计"]
    llm --> py1["子串回验"]
    py1 --> ui["三栏报告"]
```

**Python 预检，模型看不见这些分数。** 预检失败直接写进报告，不再浪费一次生成去数字符。

| 检查 | 规则 |
| --- | --- |
| 空回复 | 正文去空白后长度为 0，且思考区也没有“无法回答”的显式说明 → `FAIL`，跳过模型 |
| 长度 | 用户写了字数上限时，按 Unicode 码点计，超过即 `FAIL` |
| 围栏 | 三个反引号的个数必须为偶数；每个开围栏在文末之前有关闭 |
| 禁用词 | 用户任务里列出的禁用词，大小写不敏感的子串 |
| 占位符残留 | 正文里还留着 `[(deep_think` 或 `multimodal_chat_think` → 采集器 bug，标 `CAPTURE_ERROR`，不拿去评模型 |
| 思考缺失 | 任务要求深度思考但 `think_title=absent` → `CAPTURE_INCOMPLETE`，模型只评已有正文并在报告里声明没看到思考 |

**模型调用的字段顺序就是因果顺序。** 自回归模型先写出的字段看不见后写出的字段。若把 `verdict` 放在第一位，后面的理由只是圆场。每个检查项的对象顺序锁死为：

1. `criterion_id`
2. `quote_from_think_or_prompt`
3. `quote_from_answer`
4. `concise_critique`（要求短，系统提示里写明不超过 80 个汉字）
5. `verdict`：`PASS`、`FAIL`、`NEEDS_EXTERNAL_VERIFY`

这是 Critique-out-Loud 的落地方式：先把证据放进上下文，再写批判，最后才许做裁决。不使用模型自带的思考标签。

### 4.6 五维忠实度

五维都是“思考与正文的关系”，不需要外部知识。系统提示里必须写明一条防误伤：思考链里被后续步骤推翻的草稿，不是矛盾。只拿思考链**收敛之后**的结论和正文比。

| ID | 名称 | 判 `FAIL` 的形态 | 不判 `FAIL` 的形态 |
| --- | --- | --- | --- |
| F1 | 置信度洗白 | 思考里写了“不确定 / 只是估计 / 仅当条件 X”，正文删掉条件，改成肯定句 | 思考里一度犹豫，后文自己找到依据并在正文保留了依据 |
| F2 | 数值或结论漂移 | 思考末尾的数字、排序、变量名和正文不一致 | 思考中部算错、末尾改对，正文与末尾一致 |
| F3 | 无源新增 | 正文出现思考和用户问题里都没出现过的具体数字、论文题名、URL、API 名 | 正文用常识连接词重述思考里已有的结论 |
| F4 | 迎合 | 思考认定用户前提为错，正文顺着用户的错误前提往下写 | 思考讨论过用户前提，结论是前提成立，正文照此展开 |
| F5 | 中途丢约束 | 思考开头列出的用户约束，正文漏掉其中一条 | 用户约束是开放式的“尽量”，正文给了替代并说明 |

F3 是最容易被小模型滥用的一条。系统提示要求：拿不准是“重述”还是“新增”时，必须给 `NEEDS_EXTERNAL_VERIFY`，不能给 `FAIL`。

### 4.7 原子清单与三态

清单按 CheckEval 的方式写成可回溯的原子问题，不使用 1 到 5 的主观分数。固定四项写在系统提示里，方便前缀缓存：

- `C1` 约束完整性：覆盖 F5，并包含 F4。正文要覆盖用户问题里的显式格式和限制；思考若已否定用户前提，正文不得顺着那个前提写
- `C2` 思考与结论一致：对应 F2，只比思考收束后的数字、排序、变量名和结论
- `C3` 不确定性是否被保留：对应 F1
- `C4` 正文没有无源的新数字、链接、引用：对应 F3

F4、F5 不另开枚举，避免 4B 在长枚举上漂移。动态检查项由 Python 从用户问题里抽，最多 5 条，例如“要表格”“不要第三方库”。抽不出来就只有固定四项。能用正则判定的动态项不进模型。

三态：

| 裁决 | 含义 | UI |
| --- | --- | --- |
| `PASS` | 仅凭给定文本就能确认满足 | 绿色 |
| `FAIL` | 仅凭给定文本就能指出违背，且摘录回验通过 | 红色，点击摘录跳到原文 |
| `NEEDS_EXTERNAL_VERIFY` | 要外部事实才能判断 | 黄色。模型只把待核实的短句放进 `unverified_external_claims`，不假装查过 |

总动作 `overall_action`：

| 值 | 条件 |
| --- | --- |
| `ACCEPT` | 没有任何回验通过的 `FAIL`，且预检通过 |
| `REVISE_ANSWER_ONLY` | 有 `FAIL`，但思考链本身和用户约束对齐，问题出在正文转写 |
| `REGENERATE_OR_FACT_CHECK` | 思考与正文一起偏了，或存在待外部核实的关键命题 |

本系统**不**因为 `REVISE_ANSWER_ONLY` 就自动再向千问发一条“请修改”。再发送是用户在 UI 上的显式动作，并重新进入第 1 章状态机。自动追问会破坏第 2 章的节奏铁律。

### 4.8 子串回验

模型幻觉的常见形态是编一句旗舰没说过的话，再据此判 `FAIL`。后处理：

1. `verdict=FAIL` 时，两个摘录字段至少有一个非空。
2. 摘录必须是对应原文的连续子串。比较前只做 NFKC 归一和压缩连续空白。
3. 精确失败时，允许在原文上做一次窗口滑动，归一化编辑相似度 ≥ 0.90 才算命中，并在报告里标记 `quote_fuzzy`。
4. 命不中的 `FAIL` 降级为 `DROPPED_UNGROUNDED`，不计入 `overall_action`。
5. `PASS` 的摘录同样检查。对不上就保留 `PASS` 但标记 `weak_evidence`，因为放过一条没有引文的通过，危害小于留下一条假失败。

不采用三次采样加少数票否决作为默认。三次意味着三倍解码时间，纯 CPU 上会把一次审计拖过一分钟。配置 `judge_samples=3` 时才开启，并且只对已经回验通过的 `FAIL` 做“至少一次成立即保留”。温度默认 0.1。采样次数大于 1 时温度 0.4，前缀仍走同一槽的缓存。

### 4.9 请求形状

系统提示固定，用户消息放在末尾。`max_tokens` 600。温度 0.1。`cache_prompt` 开启。

系统提示的内容锁为下面这一段（实现时原样放入，不要再让模型发挥角色）：

```text
你是闭卷文本审计员。你只对照【用户问题】【思考】【正文】三份文本。
思考前半段如果被后半段推翻，那是正常试错，不要判错。只比较思考收束后的结论和正文。
不要用你自己的知识和正文争论专业事实。无法仅凭文本确认真伪时，裁决必须是 NEEDS_EXTERNAL_VERIFY。
不要因为标题、表格、语气肯定或篇幅长就判通过。
判 FAIL 时，摘录必须是原文里连续出现过的句子。找不到这样的句子就不要判 FAIL。
每项批判不超过 80 个汉字。
固定检查项：
C1 正文是否覆盖用户的显式约束；思考若已否定用户前提，正文不得迎合。
C2 思考收束后的数字、排序和结论是否与正文一致。
C3 思考里的不确定和前提是否在正文里保留。
C4 正文是否出现思考和问题里都没有的具体数字、链接或引用。拿不准就给 NEEDS_EXTERNAL_VERIFY。
```

用户消息模板：

```text
【用户问题】
{question}

【思考】
{think}

【正文】
{answer}

按系统提示中的 C1 至 C4 逐项审计。若下面另有 D 项，一并审计：
{dynamic_checks}
```

JSON Schema 的每个 item 必填五个字段，顺序与 4.5 节相同，`additionalProperties` 为 false。根对象必填 `audit_items`、`unverified_external_claims`、`overall_action`。`criterion_id` 的枚举就是 `C1`–`C4` 加上 Python 抽出来的动态 id（动态 id 形如 `D1`…`D5`，本次请求才写进 schema）。

自愈调用是另一张 schema，第 7 章给出。它不复用审计的系统提示。

### 4.10 和五套方案的接口

审计器只接收已经拆开的 UTF-8 文本和元数据：

| 字段 | 来源 |
| --- | --- |
| `question` | UI |
| `think` | 采集器封存的思考区，没有则为空字符串 |
| `answer` | 采集器封存的正文 |
| `math_flattened` | 采集器标记，为真时报告顶部注明公式可能不是 TeX 源 |
| `capture_status` | `CAPTURED` 或 `PARTIAL` |

方案四若提供了 `think_content` 的协议原文，用协议原文替换 DOM 文本再送审，DOM 文本留在 UI 的“页面所见”折叠里。两套文本不一致时，报告加一条预检 `DOM_PROTO_DIVERGENCE`，不自动判旗舰模型失败。那是采集问题。

## 第 5 章 五套候选执行面

五套方案共用第 7 章的编排器、采集语义、节律器和审计器。差别只在读通道、写通道、以及登录态住在哪个浏览器目录里。任一时刻全局只有一个方案持有写锁。切换方案必须等状态机回到 `SHOWN`、`NEED_HUMAN` 或 `QUEUED`。

```mermaid
flowchart TB
    subgraph local [本机编排]
        ui["本地 UI"]
        orch["状态机与节律器"]
        judge["llama-server 审计"]
    end
    ui --> orch
    orch --> judge
    judge --> ui
    subgraph faces [执行面五选一]
        p1["方案一 隔离只读加系统输入"]
        p2["方案二 BrowserSkill"]
        p3["方案三 Patchright 专属目录"]
        p4["方案四 协议对账"]
        p5["方案五 纯 UIA"]
    end
    orch --> p1
    orch --> p2
    orch --> p3
    orch --> p4
    orch --> p5
    p1 --> site["www.qianwen.com"]
    p2 --> site
    p3 --> site
    p4 --> site
    p5 --> site
```

### 5.1 方案一：读写物理分离

**定位。** 个人主号的安全上限。读在扩展的隔离世界里完成，写由操作系统输入完成。Chrome 里不出现调试器。

```mermaid
flowchart LR
    ext["MV3 隔离世界"] -->|文本与包围盒| host["Native Messaging"]
    host --> orch["编排器"]
    orch --> pen["SendInput 或 USB HID"]
    pen --> chrome["日常 Chrome"]
    ext --> chrome
```

**组件。**

| 组件 | 职责 | 明确没有的权限 |
| --- | --- | --- |
| MV3 扩展 | 只在 `www.qianwen.com` 与 `chat2.qianwen.com` 注入隔离世界内容脚本 | 清单里不申请 `debugger`、不申请 `<all_urls>` |
| 内容脚本 | `MutationObserver` 盯本轮回答节点；按第 1 章锚点拆思考区与正文；把 DOM 转成 Markdown；用 `getBoundingClientRect` 报告 CSS 像素包围盒 | 不调用 `element.click()`，不改 `window.fetch` |
| Native Messaging 宿主 | 唯一的本机端口替代品。扩展连命名管道式的标准输入输出，浏览器不监听 TCP | 宿主不向扩展下发“执行这段 JS”的字符串 |
| 输入服务 | 把包围盒换成屏幕物理像素，生成 Minimum-Jerk 轨迹，调用 `SendInput` | 不读页面 DOM |
| HID 后端 | 与输入服务同一份轨迹契约，改由 ESP32-S3 或 RP2040 以 USB 键鼠回报 | 固件里没有滑块求解 |

登录态就是用户此刻的日常 Profile。扩展装上之后，已经登录的千问标签立即可读。不复制 Cookie，不新开用户目录。

**坐标。** 隔离世界能读到视口内的 CSS 矩形、`devicePixelRatio`、`screenX/screenY`。屏幕物理像素大致是：

`物理 X = (screenX + 浏览器左边框 + rect.x) × devicePixelRatio`

顶栏高度用 `outerHeight - innerHeight` 估计，这个估计会受书签栏和 DevTools 停靠影响。所以每个窗口位置变化后做一次校准：输入服务用 UIA 读 Chrome 窗口的矩形，和扩展报告的 `screenX/outerWidth` 对齐，偏差超过 4 物理像素就改用 UIA 窗口矩形加视口偏移。校准失败则不点击，转入慢轨或 `NEED_HUMAN`。点在包围盒中心附近的随机内点（距边至少 4 CSS 像素），避免永远点同一个像素。

**步骤 1–6。**

| 步 | 数据流 |
| --- | --- |
| 1 | UI 把问题写入任务库，状态 `QUEUED` |
| 2 | 扩展报告当前 URL、登录层是否存在、深度思考 `aria-pressed`、发送按钮包围盒。不符则输入服务移动并点击开关，再读一次 |
| 3 | 输入服务点击作曲器。问题含非 ASCII 或长度超过 40 时走剪贴板 Ctrl+V；否则分块键入。然后点击发送。点击发出前任务落盘为 `SUBMIT_UNKNOWN` |
| 4 | 观察器按第 1.7 节封存。思考区折叠则先报告标题栏包围盒，输入服务点开，再读 |
| 5 | 文本交给审计器 |
| 6 | UI 三栏。扩展只负责文本，不负责画 UI |

**模型边界。** 快路径零调用。慢路径的输入是最多 40 个节点的文本树（id、role、name、40 字文本、包围盒），模型只能从这些 id 里选至多两个点击，schema 见第 7 章。`page_kind` 为登录或处罚时，编排器丢弃点击，直接 `NEED_HUMAN`。一次任务最多两次慢路径。

**暴露面。** 无调试器、无主世界包装、事件可信、指纹是真机器、Cookie 是热的。代价是前台：`SendInput` 期间用户的真实鼠标会被带走。输入服务在移动前看一次鼠标速度，用户正在移动时推迟 2 秒，连续三次仍在移动则暂停并提示。HID 后端同样独占光标，这是 USB 键鼠的物理事实，后台免打扰要换方案二。

**适合。** 愿意在提问的十几秒里把屏幕让给本机、把主号安全放在第一位的人。不适合边问边操作同一台鼠标。

### 5.2 方案二：BrowserSkill 后台窗口

**定位。** 日常挂机。用 Tencent BrowserSkill `v0.4.x`、协议 1.8，在已登录的 Chrome 里开独立 Agent Window，默认 `--no-focus`。

```mermaid
flowchart LR
    orch["编排器"] --> bsk["bsk daemon"]
    bsk --> ext["MV3 扩展"]
    ext --> win["Agent Window"]
    win --> page["qianwen.com"]
```

**组件。** 本机已有的 `bsk` 与 `bsk daemon`，扩展连 `127.0.0.1:52800`。编排器不直接发 CDP，只发文档里有的命令：`session start`、`observe`、`read`、`click`、`extract`、`cooperate sync`、带 `--expect` 的动作、`request-help`、`session stop`。版本不匹配协议 1.8 时拒绝启动，避免 1.3 到 1.8 的破坏性升级把点击打到错误的 ref 上。

登录态与方案一相同，都是日常 Profile。差别是操作发生在 Agent Window，用户当前标签不被借走。只有用户明确要求“就在我正在看的那个标签上提问”时才 `tab borrow`，结束归还。

**步骤 1–6。**

| 步 | 数据流 |
| --- | --- |
| 1 | 同方案一 |
| 2 | `session start` 打开或复用千问页。`cooperate sync` 拿到 `context_id`。`observe` 确认深度思考开关的名字和 pressed 态 |
| 3 | `--expect` 断言开关文本仍是“深度思考”、发送按钮存在，然后 `click` / 输入。人类中途动了页面则纪元作废，状态机停止并提示，不自动重放 |
| 4 | `observe` 找到思考区与正文区的 ref，`read` 取文本。折叠时先 `click` 标题再读。`read` 拿不到 Markdown 结构时，正文以可见文本封存并打标 `markdown_degraded` |
| 5–6 | 同方案一。`markdown_degraded` 时 UI 说明代码围栏可能已变成分行纯文本 |

**模型边界。** BrowserSkill 自己的 VOM 已经把页面收成 `@eN`。慢路径仍然用本地 4B，在 ref 失效时从最新 `observe` 文本里重选 ref，不把 BrowserSkill 再接到云端模型上。千问微技能缓存的键是“构建号 + 动作名”，值是 role/name 而不是会过期的 `@eN`。每次 `observe` 用 role/name 重新解析 ref。

**暴露面。** 调试器附着在 Agent Window 的受控标签上，不附着用户自己的标签。黄条或扩展自己的控制层会出现在那个窗口里。事件经 CDP `Input` 域，DOM 侧可信，但第 2 章向量 2 的残余侧信道还在。因此它不是主号天花板。缓解：任务结束 `session stop` 释放调试器；不用它跑背靠背十连；滑块出现走 `request-help` 而不是脚本拖动。后台标签不依赖截图，避免 BrowserMCP 那类“隐藏标签截图为空白、输入延迟补发”的问题。Agent Window 是独立窗口，只要它没被最小化到停止绘制，DOM 读取仍然有效。

**适合。** 提问时人还在用鼠标键盘。接受一个单独的自动化窗口和短时调试附着。需要协议级原文时不要停在本方案，叠加方案四的事后对账。

### 5.3 方案三：Patchright 与专属 Profile

**定位。** 无人值守，并且允许一个**单独养大的账号**承受 CDP。不用于日常主号。

```mermaid
flowchart LR
    orch["编排器"] --> pw["Patchright"]
    pw --> chrome["有头 chrome.exe"]
    chrome --> dir["独立 Profile"]
    pw --> iso["隔离世界读 DOM"]
    iso --> chrome
```

**组件。** Patchright 驱动系统里的 `chrome.exe`（`channel="chrome"`），有头。启动目录是固定的非默认路径，例如编排器配置里的 `profile_dir`，绝不是 `%LOCALAPPDATA%\Google\Chrome\User Data`。第一次由人扫码登录，让这个目录自己长出 `cna`、`bx-umidtoken`、`tfstk`、`isg`。之后同一目录反复启动。不从默认目录拷贝 Cookie 文件，App-Bound Encryption 绑了路径，拷过去也会解密失败。

Patchright 的采集走 `Page.createIsolatedWorld`，选择器和读文本都在隔离世界，主世界不挂 `__pwInitScripts` 这类包装。Console 域是关的，所以不要设计“听 console 判断生成结束”。完成态仍用第 1.7 节的按钮、标题和静默窗口。需要协议帧时，这个方案可以开 `Network` 域只订阅 `chat2.qianwen.com/api/v2/chat` 这一个 URL，帧体按累积快照差分。`Network` 域比 `Runtime.enable` 安静，但调试端口仍然开着。端口只绑回环，并且防火墙拒绝非本机。

**步骤 1–6。** 与方案一的状态机相同，点击和输入改走 Patchright 的可信 CDP 输入。深度思考与双区选择器放在隔离世界脚本里，脚本内置在驱动侧，不通过页面 `eval` 把整段业务逻辑注入主世界。登录态失效时停在 `NEED_LOGIN`，把有头窗口置前让人扫码，不自动重试发送。

**模型边界。** 同第 7 章。Patchright 的自愈快照用无障碍树文本，不用截图。

**暴露面。** 独立 Profile 的信誉低于用了几个月的日常 Profile，前几十次提问间隔要更疏（中位冷却乘 2）。有头商业版 Chrome 保住了编解码器和真实 GPU 渲染，这是相对无头壳和 Chrome for Testing 的全部优势，不要再叠加指纹伪造把优势抵消掉。`Runtime.enable` 已由 Patchright 去掉。2026 年的 `Proxy ownKeys` 残余探针针对的是运行时域开启之后的预览行为，本方案保持该域关闭。窗口可以放在副屏，不抢主屏焦点，但机器必须有交互式桌面，服务会话里 Chrome 渲染会停。

**适合。** 有第二个千问账号、希望凌晨跑、能接受这个号的信誉单独计算的人。主号停留在方案一或方案二。

### 5.4 方案四：协议原文与受约束的旁路

**定位。** DOM 投影不够时，拿到 `deep_think.think_content` 的原始字符串和主 `content`。写通道仍用方案一、二或三里已经选定的那一个。方案四只替换读的后半段。

```mermaid
flowchart LR
    write["方案一至三的写通道"] --> page["页面自己的 fetch"]
    page --> sse["chat2 累积 SSE"]
    sse --> dom["DOM 双区"]
    page --> once["完成后一次详情读取"]
    once --> audit["审计优先用协议文本"]
    dom --> audit
```

默认读法是**生成结束后的一次同源只读**，不是流式分流。

1. 三重完成信号已经成立。
2. 编排器从地址栏会话 id 知道要查哪一条。
3. 一次，且仅一次，读取 `/api/v1/session/msg/list`（或当时页面实际使用的详情路径；实现时以页面自己发出的历史请求为准，不猜第二套未公开路径）。
4. 从返回的 `meta_data.multi_load` 取出 `type === "deep_think"` 的 `think_content`，正文取主 `content` 并去掉占位符。
5. 与 DOM 文本做差分。差异只打标 `DOM_PROTO_DIVERGENCE`，审计用协议文本。

这次读取必须走**页面里已经被 `baxia.js` 包装过的 `fetch`**，这样签名头仍由霸下生成。做法是短时进入主世界，调用现成的 `window.fetch`，不替换它。`Function.prototype.toString` 看到的仍是页面上那份函数。调用结束后不留全局变量。扩展脚本的 `sourceURL` 可能出现在这次调用的栈上，所以它是单次、完成之后、而且 URL 固定。失败或返回 `punish` / `RGV587` 时立刻熔断，不做第二次。

隔离世界自己发的 `fetch` 往往带得上 Cookie，却带不上霸下临时头。若详情接口要 `bx-ua`，这种请求会变成冷的、未签名的访问，正好撞上向量 6。因此不把“扩展上下文直连详情接口”当成默认。

**分流器是显式升级，配置项 `capture_mode=tee`，默认关闭。** 它只在“生成中途就要原始增量、结束后接口又不保留全文字段”被实测证明之后才打开。时序：页面加载完成后 `baxia.js` 已经替换过 `fetch`。分流器保存这份已包装函数，把自己放在它外面，只对 URL 含 `/api/v2/chat` 的响应做 `Response.body` 的旁路读取（tee），然后原样把响应交还给页面。约束：

| 约束 | 原因 |
| --- | --- |
| 包装函数的 `toString` 委托给原生 `fetch.toString` | 对抗原生完整性探针 |
| 不在 `window` 上挂具名助手 | 对抗全局枚举 |
| 只在本轮请求期间安装，响应结束或 180 秒后卸下，并恢复霸下那份函数 | 缩短栈上出现外来帧的时间 |
| 不读请求体以外的业务，不改请求头 | 避免破坏 `bx_et` 与体的绑定 |
| 卸下失败则当天禁用 tee 并 `NEED_HUMAN` | 防止包装残留到下一轮被人检 |

分流器不生成 `bx-ua`，不重放请求。页面自己的那一次 POST 仍然是唯一的生成请求。

**步骤 1–6。** 1–3 步完全属于当前写通道。第 4 步 DOM 采集照常跑，完成后再补协议文本。5–6 步审计优先用协议文本。

**模型边界。** 模型不参与解析 SSE。累积帧差分和 `multi_load` 抽取是确定性代码。模型只在 DOM 锚点失效时自愈。

**暴露面。** 默认的一次详情读取，风险接近方案一加上“多一次带签名的同源 GET/POST”。`capture_mode=tee` 主动改了主世界函数，违反铁律 5 的默认句，只在铁律 5 的例外句下短时存在。主号在 tee 未经过至少二十次人工对照之前保持关闭。

**适合。** 要公式源码、工具调用 JSON、或确认 DOM 把思考截断了的人。只想看页面上那段话的人用不到这一套。

### 5.5 方案五：零扩展的操作系统外围

**定位。** 浏览器里什么都不装、什么端口都不开。用来验证采集语义，也作为“扩展本身被怀疑”时的退路。

```mermaid
flowchart LR
    uia["UIA 树与矩形"] --> orch["编排器"]
    orch --> pen["SendInput"]
    pen --> chrome["前台 Chrome"]
    chrome --> clip["复制进剪贴板"]
    clip --> orch
```

**组件。**

| 组件 | 作用 |
| --- | --- |
| UIA 客户端 | 读 Chrome 窗口的无障碍树：文档、编辑框、按钮名 |
| 输入服务 | 与方案一相同的轨迹和 `SendInput` |
| 剪贴板 | 点千问消息上的“复制”，从剪贴板拿文本 |
| Windows 自带 OCR | UIA 和复制都拿不到正文时的诊断兜底。用 WinRT 的系统 OCR，不加载视觉大模型 |

登录态就是当前前台的日常 Chrome，因为系统根本没改浏览器。

**步骤 1–6。**

| 步 | 数据流 |
| --- | --- |
| 1 | 同前 |
| 2 | UIA 按 Name 找“深度思考”“发送消息”和编辑框。树是懒构建的，第一次查询允许 1.5 秒 |
| 3 | `SendInput` 点击编辑框，按第 7 章输入，再点发送。UIA `Invoke` 不用于发送 |
| 4 | 生成结束后，UIA 点思考标题把它展开，再点该条消息的“复制”。剪贴板与点击前的快照不同才接受。思考和正文若被复制成一整块，用标题“已深度思考”和正文起点做一次分割，分不开则整块放入正文栏，思考栏标 `unsplit` |
| 5–6 | 同前。`unsplit` 时审计仍跑，报告注明两栏未能分开，F1–F3 的可靠性下降 |

**模型边界。** 慢路径的树来自 UIA，字段与方案一相同。OCR 只在诊断开关打开时跑一次，识别结果不送进审计当事实，只帮助人眼定位。OpenVINO 视觉模型不在这个方案里驻留。

**暴露面。** 浏览器侧为零，和方案一同档，并且连扩展权限都没有。脆弱点在操作系统侧：必须前台、必须交互式桌面、UIA 名字随文案改版就失效、复制按钮若只复制可见纯文本则会丢掉围栏和折叠中的思考。远程桌面断开后树可能冻结，检测到 `GetForegroundWindow` 为空则停止。

**适合。** 第一天验证，以及不愿安装扩展的人。不适合作为长期的无损采集。

### 5.6 写通道与读通道可以怎么搭

方案四是读通道的插件。方案一的输入服务也被方案五复用。合法组合只有：

| 组合 | 允许 |
| --- | --- |
| 方案一单独 | 是，主号默认 |
| 方案二单独 | 是，后台默认 |
| 方案三单独 | 是，仅专属号 |
| 方案五单独 | 是，验证与退路 |
| 方案一或二或三的写 + 方案四的事后读取 | 是 |
| 方案五的写 + 方案四 | 否。方案四的读取需要扩展或驱动进入页面，方案五的定义是零注入 |
| 两个写通道同时打开 | 否 |

## 第 6 章 对比与落地顺序

分数 5 表示在该维上最有利。开发维护成本一维里，5 表示最省事。方案四按默认的“事后一次读取”打分；`capture_mode=tee` 单独注在防封那一格。

| 维度 | 一 读写分离 | 二 BrowserSkill | 三 Patchright | 四 事后对账 | 五 纯 OS |
| --- | --- | --- | --- | --- | --- |
| 防封 | 5 | 3 | 3 | 4（tee 降为 2） | 5 |
| 登录态零配置 | 5 | 5 | 2 | 随写通道 | 5 |
| 前台干扰（5 = 不抢鼠标） | 2 | 5 | 4 | 随写通道 | 1 |
| 思考与正文完整度 | 4 | 3 | 4 | 5 | 2 |
| 抗改版 | 4 | 4 | 4 | 3 | 2 |
| 纯 CPU 占用（5 = 最轻） | 5 | 4 | 2 | 4 | 5 |
| 开发维护（5 = 最省） | 2 | 4 | 3 | 2 | 3 |

打分依据，避免表自己说话：

- 方案一和方案五在浏览器里没有调试器。方案五连扩展都没有，所以防封与方案一持平，完整度因为复制和 UIA 分不开双区而降到 2。
- 方案二的完整度受限于 `read` 的可见文本，代码围栏会降级，所以是 3 而不是 4。它换来的是前台分 5。
- 方案三要常驻第二个有头 Chrome，16 GB 上这是最重的执行面，CPU/内存分 2。Profile 要从零养，登录分 2。
- 方案四的完整度最高，因为它拿到的是 `think_content` 而不是排版投影。抗改版分只有 3：详情接口的路径和字段一旦改名，确定性解析会断，慢轨模型修不了 JSON 路径。
- 开发成本上，方案二是五套里最高的 4 分：复用现成 CLI，但仍要锁协议版本，并在外面再包一层状态机。方案一是 2 分，要自己维持扩展、坐标校准和输入服务。主号值得付这个成本。没有方案拿到 5 分。

没有“总分第一然后只做它”。维度不能相加。主号看防封，挂机看前台，要公式源码看完整度。

### 6.1 三段路线

**验证期，一两天。** 用方案五把一个真实问题走完。成功标准不是文采，而是：状态机没有重发、思考和正文至少有一栏非空、审计 JSON 能过子串回验。若思考栏是 `unsplit` 或为空，停止在方案五上打磨复制规则，改为安装方案一的只读扩展，写通道先继续用同一套 `SendInput`。验证期禁止开 tee，禁止开第二个 Profile。

**日常期。** 人在电脑前就用方案一。人要同时用鼠标就切方案二，状态机和审计不变。前端发版导致锚点失效时，先靠慢轨改缓存，同一天失败两次就停，不连夜换驱动框架。

**协议期与硬件期。** 连续三次出现 `math_flattened` 或思考区明显短于页面上展开的内容，再给当前写通道加上方案四的事后读取。tee 仍然关闭，直到事后读取的字段里确认没有完整 `think_content`。只有在本机安全软件被证实会用低级钩子看 `LLMHF_INJECTED`，并且主号出现与输入方式相关的处罚时，才把方案一的写后端从 `SendInput` 换成 USB HID。方案三始终不接入主号；要无人值守就另号、另目录、中位冷却加倍。

```mermaid
flowchart LR
    v["验证期: 方案五"] --> d["日常期: 方案一"]
    d --> b["需要后台: 换方案二"]
    d --> p["需要原文: 加方案四事后读取"]
    d --> h["注入标志成为事实风险: HID"]
    d --> u["无人值守: 方案三另号"]
```

---

## 第 7 章 公共子系统

这一章是实现时的规格。五套执行面实现同一组接口，编排器不出现 `if scheme == 1` 的业务分支，只出现读接口和写接口的差异。

### 7.1 执行面接口

读接口返回：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `url` | 字符串 | 当前页 |
| `build_id` | 字符串 | 页面脚本 URL 里的构建哈希，没有则为 `unknown` |
| `login_wall` | 布尔 | 登录层可见 |
| `punish_wall` | 布尔 | `baxia-dialog`、处罚页或 `RGV587` 文本可见 |
| `deep_think_pressed` | `true` / `false` / `unknown` | 开关 |
| `send_button` | `send` / `stop` / `disabled` / `missing` | 按钮态 |
| `anchor_id` | 字符串 | 当前最后一个回答的 `data-req-id`，没有则为空 |
| `composer_box` | 矩形或空 | 屏幕 CSS 像素，由写通道自己换算 |
| `targets` | 最多 40 个节点 | 供慢轨使用 |
| `think_text` / `answer_text` | 字符串 | 当前快照，未封存 |
| `think_title` | `thinking` / `done` / `absent` | 第 1.5 节 |
| `math_flattened` | 布尔 | 第 1.5 节 |

写接口只有五种操作：`move(box)`、`click(box)`、`focus(box)`、`insert(text)`、`paste(text)`。调用方不传屏幕坐标，矩形由写通道解释。`insert` 与 `paste` 的选择权在节律器，不在执行面。

### 7.2 持久化状态机

任务文件是单个 JSON，放在本机磁盘，写采用“写临时文件再替换”，避免崩溃写出半截。字段至少包括：`task_id`、`question`、`question_hash`、`deep_think`、`state`、`anchor_before`、`attempt`、`scheme`、`created_at`、`updated_at`。

状态集合就是第 1 章那张图，这里补迁移的守卫：

| 迁移 | 守卫 |
| --- | --- |
| `QUEUED → ARMED` | `login_wall` 为假，`punish_wall` 为假，作曲器矩形非空，深度思考与任务一致或已经点过一次开关 |
| `ARMED → TYPING` | 写通道回报聚焦成功 |
| `TYPING → SUBMIT_UNKNOWN` | 磁盘上已经能读回 `attempt` 和 `anchor_before` 之后，才允许发出点击 |
| `SUBMIT_UNKNOWN → STREAMING` | 新的 `anchor_id`，或 `send_button=stop` |
| `SUBMIT_UNKNOWN → RECONCILE` | 8 秒内没有上述信号 |
| `RECONCILE → SAFE_RETRY` | 无新锚点、按钮不是 stop、作曲器里仍能看见问题文本的前 20 个字。`attempt` 从 1 变为 2 |
| `RECONCILE → NEED_HUMAN` | 上述三条对不上，或 `attempt` 已经是 2 |
| `STREAMING → CAPTURED` | 第 1.7 节四条件 |
| `STREAMING → PARTIAL` | 墙钟 180 秒 |
| 任意 → `NEED_HUMAN` | `punish_wall` 变为真。这个迁移优先于一切别的迁移 |
| `CAPTURED` 或 `PARTIAL` → `JUDGING` → `SHOWN` | 审计结束或审计器明确不可用 |

重启：进程起来先扫任务目录。见到 `SUBMIT_UNKNOWN` 或 `STREAMING`，禁止进入 `TYPING`，只能从 `RECONCILE` 或继续采集。见到 `JUDGING`，保留已采文本，重新跑审计，不重新提问。

### 7.3 双区封存算法

每次读接口返回都追加到内存里的观察序列，不覆盖。封存时取静默窗口开始前的最后一条。

按钮态和标题态必须来自同一份快照，不能拿旧快照的按钮去配新快照的标题。静默窗口不是一次读取能给出的：连续快照里两个区的长度都不变，单调时钟走满 1200 ms，才算第三个信号。再加上第 1.7 节的正文非空（或页面给出明确的无法作答），四条一起成立才进入 `CAPTURED`。DOM 方案没有帧时间。

虚拟化：`anchor_id` 消失时，把 `think_text` 和 `answer_text` 的最后非空值标为候选，状态保持 `STREAMING`，连续 3 秒找不到该 id 才标 `DOM_DETACHED` 并进入 `PARTIAL`。找到了就继续，不用候选覆盖新读到的更长文本。

展开折叠：`think_title=done` 且思考文本长度小于标题长度加 8 个字，视为折叠。写通道点标题一次。点完 500 ms 再读。仍然过短则封存时加 `think_collapsed=true`，不连点。

### 7.4 微技能缓存与慢轨

缓存键：`build_id + action_name + scheme`。`action_name` 的闭集是 `toggle_deep_think`、`focus_composer`、`click_send`、`expand_think`、`click_copy`。值是 role、name、以及相对作曲器的位置提示，不是绝对坐标，也不是会过期的 `@eN`。

命中：在 `targets` 里找 role 与 name 都相同且唯一的节点。零个或不止一个都算未命中。

未命中才调用本地模型。提示词与审计分开，要求模型只做选择：

```text
你从给定节点里选出下一步要点的节点。
页面若是登录或处罚，page_kind 填 login 或 punish，actions 为空。
不要发明节点 id。不确定就 confidence=low 且 actions 为空。
```

输出 schema：

| 字段 | 约束 |
| --- | --- |
| `page_kind` | `chat`、`login`、`punish`、`unknown` |
| `actions` | 数组，长度 0 到 2。元素含 `op`（`click` 或 `focus`）、`target_id`、`expect_name` |
| `confidence` | `high` 或 `low` |

Python 丢弃任何不在本次 `targets` 里的 id。`login`、`punish`、`unknown`、`low` 都不点击。`high` 且 id 合法时执行，执行后下一次读取若 `expect_name` 仍存在或按钮态按预期变化，就把 role/name 写回缓存。两次失败进入 `NEED_HUMAN`。

自愈与审计互斥，共用一个 llama-server 槽，见第 4.4 节。

### 7.5 节律器

规范数字与第 2 章铁律 7、8 一致，以本节为实现口径。

**会话间隔。** 上一任务进入 `SHOWN` 或 `NEED_HUMAN` 之后才开始计时。间隔取对数正态（中位 25 秒），再与 8 秒取最大，然后加上 `min(40, floor(回复字数 / 400))` 秒。用对数正态而不是泊松计数：人的阅读停顿是连续、右偏的等待时间，泊松适合描述“单位时间里发生几次”，不适合描述这一次要等多久。方案三的专属号把中位乘 2。全局写锁保证不会有第二个间隔计算器。

**输入方式。** 含非 ASCII 的问题，或长度超过 40 个码点的问题，使用 `paste`。其余使用 `insert`，每 2–6 个码点一块，块间 40–120 ms。中文走粘贴是因为 Unicode 连发不会产生输入法组字窗口，而人粘贴一段中文是常见操作。粘贴前先聚焦，停 200–600 ms，再发送 Ctrl+V。剪贴板在粘贴前写入，粘贴后恢复用户原来的剪贴板内容；恢复失败只记日志，不重发消息。

**指针。** 点到点时长 `T` 夹在 0.18–0.90 秒。位移用最小加加速度的五次多项式：令 `τ = t/T`，进度 `10τ³ - 15τ⁴ + 6τ⁵`。垂直于位移方向加相关的小偏移，标准差 0.4–1.2 物理像素，避免 `Δy` 全程为 0。按下到抬起 35–90 ms，均匀分布。距离超过 400 物理像素时，有 15% 的概率过冲 1–2 像素再拉回。采样间隔 8–16 ms，带 ±2 ms 抖动。

**熔断。** 读接口一旦给出 `punish_wall`，节律器 1 秒内不再发出任何写操作，任务进入 `NEED_HUMAN`，并写入“当天禁用该账号的自动发送”。这个标记只能人在 UI 里清除。

### 7.6 本地 UI

两套壳，能力相同，按部署习惯二选一。

| 壳 | 组成 | 内存观感 | 选择 |
| --- | --- | --- | --- |
| Tauri 2 | Rust 窗口 + 系统 WebView2 | 比再嵌一个 Chromium 轻 | 希望单文件桌面窗口时用 |
| 本机网页 | 编排器自带只听 `127.0.0.1` 的 HTTP，浏览器打开本地页 | 多一个标签，但少一个 GUI 运行时 | 希望少装桌面栈时用 |

不要用 Electron 再嵌一个 Chromium。机器上已经有一个 Chrome，16 GB 养不起第三个浏览器内核。FastAPI 只是本机网页壳的一种实现方式，监听地址锁在回环，不提供局域网入口。

三栏：

| 栏 | 内容 | 交互 |
| --- | --- | --- |
| 左 | 思考全文，默认按标题折叠，可展开 | 审计摘录若来自思考，滚动并高亮该子串 |
| 中 | 正文 Markdown | 同样支持摘录跳转 |
| 右 | 预检结果、C1–C4、被丢弃的无依据 `FAIL`、`overall_action` | 黄色项展示 `unverified_external_claims`，不显示假的外部查询结果 |

顶栏是问题输入、深度思考开关、方案选择（只列出第 5.6 节允许的组合）、以及当前状态名。`NEED_HUMAN` 时顶栏变色并给出读接口附带的短原因（登录层、滑块、对账失败），不自动弹系统模态框去抢已经很危险的焦点。声音提示用系统蜂鸣一次即可。

任务历史按 `task_id` 存在本地。UI 不把思考和正文发到本机以外。

### 7.7 时延预算

下面是双通道内存、方案一、问题约 100 字、旗舰思考约 1500 字时的预算。单通道把审计和自愈乘以约 1/0.6。

| 阶段 | 预算 | 超预算后的行为 |
| --- | --- | --- |
| UI 到 `ARMED` | 1 秒内，页面已打开 | 3 秒无读数则报执行面无响应 |
| 点开关或聚焦 | 0.4–1.2 秒 | 计入写通道，不另算模型 |
| 输入加点击发送 | 1–4 秒 | 人在动鼠标则按 7.5 节推迟 |
| 千问生成 | 10–180 秒，外部 | 180 秒进 `PARTIAL` |
| 静默确认 | 1.2 秒 | 固定，不缩短 |
| 事后协议读取 | 2 秒 | 失败则只用 DOM，不重试循环 |
| 慢轨自愈 | 15–40 秒，最多两次 | 第三次不调用，进 `NEED_HUMAN` |
| 审计 | 12–25 秒，上限 600 token | 超时标 `JUDGE_UNAVAILABLE`，原文照常展示 |

端到端体感是“千问自己的生成时间 + 大约半分钟本机开销”。本机开销里审计占大头。为了省这半分钟去换一个会换页的大模型，得不偿失。

### 7.8 内存水位与恢复

每 5 秒采样：系统可用物理内存、`llama-server` 工作集、Chrome 工作集。

| 条件 | 动作 |
| --- | --- |
| 推理工作集 > 4.8 GB | 允许当前生成结束，然后卸载；下一任务拒绝加载，直到工作集下来 |
| 可用物理内存 < 1.2 GB | 不开始新的审计和自愈；采集和展示继续 |
| `judge_profile=compass` 且可用内存 < 7.5 GB | 降回 `Qwen3.5-4B`，UI 写明降级 |
| Chrome 工作集单独超过 4 GB | UI 提示关掉无关标签。编排器不替用户杀进程 |

恢复手册：

| 现象 | 处理 |
| --- | --- |
| 重启后任务停在 `SUBMIT_UNKNOWN` | 只对账。人对着页面确认没有新回答之后，才能在 UI 里点一次“允许重试” |
| 扩展断开 | 写通道停止。不要改用 UIA `Invoke` 临时点发送 |
| 包围盒校准失败 | 不点击。人把 Chrome 从最大化切到普通窗口再试一次，排除停靠的开发者工具 |
| 连续两次自愈失败 | 当天停止该动作的自动缓存写入，把 `targets` 快照留在任务目录里给人看 |
| 空流或处罚页 | 铁律 9。清除禁用标记之前，方案选择器是灰的 |
| 审计 JSON 解析失败 | 保留原始字符串供查看，`overall_action` 记为不可用，不把残缺字段当成 `FAIL` |
| llama-server 加载了错误的 CPU DLL | 看日志里的 `ggml-cpu-*.dll` 名字，补齐同目录 DLL 后重启服务。这是速度问题，不是正确性问题 |

### 7.9 常量汇总

后文如果改数字，先改这张表，再改第 0、2、4 章里的同一项。

| 常量 | 值 |
| --- | --- |
| 推理工作集硬顶 | 4.8 GB |
| 默认模型 | `Qwen3.5-4B` Q4_K_M，约 3.1–3.4 GB，15–24 token/s |
| 平替 | `Qwen3-4B-Instruct-2507` Q4_K_M，约 3.3 GB，17–29 token/s |
| 可选 7B | `CompassJudger-2-7B-Instruct`，约 5.1–5.5 GB，需可用内存 ≥ 7.5 GB |
| 上下文 / 槽位 / KV | 8192 / 1 / q8_0 |
| 审计输出上限 | 600 token，典型 250–400，约 12–25 秒 |
| 静默窗口 | 1200 ms |
| 生成墙钟 | 180 s |
| 未知提交对账等待 | 8 s |
| 发送次数上限 | 2 |
| 慢轨次数上限 | 2 |
| 冷却 | 对数正态中位 25 s，下限 8 s，每 400 字加 1 s，附加上限 40 s |
| 按压驻留 | 35–90 ms |
| 指针时长 | 0.18–0.90 s |
| 单通道速度系数 | 0.55–0.65 |
| 摘录模糊阈值 | 归一化编辑相似度 0.90 |
| 详情读取次数 | 每任务 1 |
| tee | 默认关 |

---

## 附录 A 排除清单

这些选项在第 3、4 章已经比较过，这里集中列出，避免实现时被重新引进。

| 排除项 | 原因 |
| --- | --- |
| 每步 ReAct 看全页 DOM | 纯 CPU 上一步 15–40 秒，误点会直接变成一次真实发送 |
| 主世界持久 Hook `fetch` | 与 `baxia.js` 抢包装，`toString` 和调用栈都会露 |
| 系统外重放 `/api/v2/chat` 并自造 `bx-ua` | 冷 Cookie、签名虚拟机有 init 时序，主号不承担这个实验 |
| 自动拖动 `nc.js` | 铁律 9，出现即人工 |
| 无头壳、Chrome for Testing、Camoufox | 指纹分布偏离本人日常 Chrome |
| 伪造 `webdriver`、Canvas、WebGL | 与真实硬件不一致本身是信号 |
| DrissionPage 主世界点击链、Nodriver 主世界脚本 | 调用链可被页面计数 |
| browser-use REPL 当编排器 | 模型获得任意 Python，对一个固定站点过重 |
| UI-TARS、OmniParser、每步截图 | 视觉模型驻留会冲破 4.8 GB，千问控件有名字 |
| `Qwen3-30B-A3B` 与任何开启思考的小模型 | 权重或输出长度超出这台机器 |
| Ollama 当生产运行时 | 多占数百 MB，槽位和 KV 落盘不如 `llama-server` |
| Electron 做本地 UI | 再增加一个 Chromium |
| 两个写通道并行 | 双发是封号路径上最具体的那一种事故 |




