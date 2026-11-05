---
title: "9000:1 与 16.4%：你的一行状态提示，比整场对话还贵"
date: 2026-10-30 20:15:54 +0800
categories: ['ai']
tags: ['智能体', '可观测性', '工程实践', '成本']
slug: status-line-tax
---

## 五个字

Claude Code 的 agents 面板里，每个后台智能体旁边都有一行小字。

`正在读取 runAgent.ts`

五个词，屏幕上占不到两厘米宽。你扫一眼，知道它还活着，然后低头继续干自己的事。这是整个界面里最不起眼的一块像素，你一天可能看它两百次，一次都不会记住。

2026 年 10 月 28 日晚，一个叫 Flamerix AI 的团队把 Claude Code 的 OpenTelemetry 导出打开，按请求里的 `query_source` 字段分组统计，发现了一件不太对劲的事：那一周，标着 `agent_summary` 的请求吃掉了他们 **16.4%** 的估算费用，而主对话——那个真正在读代码、改文件、跑测试的循环——是 **15.7%**。

一行状态提示，比它所描述的那个东西还贵。

我把这件事从头捋了一遍，包括在我自己机器上装的 Claude Code 2.1.308 里 grep 了一通。它比"某个功能偷偷烧钱"这种标题要更有意思一些：**它烧钱的方式，恰好说明了 agent 工具链现在最缺的是什么。**

---

## 一、那五个字是怎么来的

先说清楚机制，因为它比想象中粗暴。

Claude Code 可以让你把智能体丢到后台跑，主会话继续干别的。为了让你知道后台那几个在干什么，面板上给每个智能体配了一行说明文字。这行字不是从日志里提取的关键词，也不是解析工具调用得到的字符串——**它是模型生成的。**

具体做法是：每隔大约 30 秒，针对每一个正在工作的后台智能体，Claude Code 把该智能体的对话 fork 一份，把**整段 transcript 原样发回给那个智能体自己用的模型**（通常是 Opus），请它"describe your most recent action in 3-5 words"。模型吐回五个词，这五个词刷新面板上的那一行。

然后这次调用的结果就被丢掉了。它不回写进那个智能体的上下文，也不进入你和 Claude 的对话。它唯一的用途，就是在屏幕上显示一小行字，三十秒后再来一遍。

我在本机二进制里核了一遍，几个关键字符串都在：

```
$ grep -a -o -F '[AgentSummary] Timer fired' claude | wc -l
2
$ grep -a -o -F 'agent_summary' claude | wc -l
14
$ grep -a -o -F 'DISABLE_NON_ESSENTIAL_MODEL_CALLS' claude | wc -l
0
```

更有说服力的是那个 fork 的调用点，直接从二进制里抠出来的：

```js
{
  cacheSafeParams: { ...r, forkContextMessages: e },
  canUseTool: y,
  querySource: "agent_summary",
  forkLabel: "agent_summary",
  maxTurns: 1,
  overrides: { abortController: s },
  skipTranscript: !0,
  ...
}
```

`maxTurns: 1` 说明它不打算行动，只回答一次。`skipTranscript: !0` 是整件事的钥匙——**这次调用不会写进磁盘上的会话日志。**

---

## 二、9000 : 1

Flamerix 的遥测里有一个数字比 16.4% 更能说明问题：`agent_summary` 这类请求的**输入 token 与输出 token 之比约为 9000 : 1**。

九千个 token 进去，一个 token 出来。这不是推理，这是复读。

GitHub issue #86879 上有另一组独立测量，量得更细。用户 rflcrz 记录了一场四小时的会话：27 个子智能体，总计 1,496 次请求，其中 **339 次**是 `agent_summary`——占原始请求数的 22.7%。这 339 次调用搬动了 **61,757,361** 个 token，产出 **21,966** 个输出 token，平均每次约 65 个。

也就是说，为了在面板上显示 339 行小字，系统反复读了六千一百万个 token 的上下文。

有人会立刻说：但提示缓存让它很便宜。没错——那 6,170 万 token 里有 6,160 万是缓存读取，这就是为什么 rflcrz 的加权占比是 18.0%，低于原始占比 22.2%。按 Opus 5.5 的公开价格（基础输入 $4/MTok、5 分钟缓存写入 $5/MTok、缓存命中 $0.20/MTok），有开发者估算：一份 10 万 token 的 transcript，命中缓存是 $0.02，未命中要按缓存写入价 $0.50 算，混合下来每次约 $0.03。三十秒一次，一小时 120 次，单个智能体一小时约 $3.55。

**缓存让它便宜，但没有让它免费。** 而且这里有个更麻烦的结构性问题：

> transcript 越长，每次摘要越贵；智能体跑得越久，transcript 越长。

三十秒一次的节奏是固定的，但每次的代价随上下文线性上涨。所以总开销是**随任务时长超线性增长**的。而"长时间运行"，恰恰是 background agent 存在的全部理由。

这个税在你最想用这个功能的场景里最贵。

---

## 三、为什么没人早点发现

一个吃掉五分之一额度的东西，按理说早该被骂上热搜。它藏了三层。

**第一层：`skipTranscript: true`。** 调用本身不落盘。磁盘上的会话日志里没有它，你翻 `~/.claude/` 翻到天亮也找不到。

**第二层：所有基于 transcript 的成本工具因此全都瞎了。** 有开发者在 2.1.308 上测了 Firekeeper 这类读 transcript 计费的工具：一场三分钟的 Haiku 会话里，约 11.9 万 token、31% 的用量在统计里直接消失了，全是摘要。另一个计费工具的维护者现在干脆在文档里警告用户——**一行不在 transcript 里，不能证明没花这笔钱。**

**第三层最要命：官方文档里有一句话。**

Claude Code 的 statusline 文档里写着：

> The status line runs locally and does not consume API tokens.

这句话本身没错，但它说的是**另一件事**——是你自己配置、跑本地脚本、显示在终端底部的那个状态栏。不是 agents 面板里那行由 Opus 生成的标签。同一份文档讲后台 token 开销的那节，压根没列这个功能；它只提到 Agent View 的行摘要用的是 Haiku 级别的小模型、几分钟刷新一次。

一个正常人读完这段，会以为自己跑的就是那个便宜版本。

顺带说一句：这两行字在 UI 上都叫"状态栏"。

---

## 四、只能开，不能关

如果说前面是信息不对称，这一部分就是设计问题。

SDK 里有个选项叫 `agentProgressSummaries`，默认值是 `false`。看起来这就是开关。但据 Flamerix 读代码的结果：**每一条写这个值的代码路径，都把它写成了 `true`**。所以你显式传 `false` 进去，什么都改变不了。

未公开的环境变量 `CLAUDE_CODE_AGENT_PROGRESS_SUMMARIES` 同理——只能启用。

刷新间隔也不在你手上。从 2.1.300（10 月 14 日）开始，间隔由服务端 flag 下发，钳在 30 到 240 秒之间，**默认值取的是下限**。没有哪个值代表"关闭"。

我在二进制里找了一圈，`DISABLE_NON_ESSENTIAL_MODEL_CALLS` 这个环境变量一处都没有——0 次匹配。它不存在。

真正能让它停下来的只有三条路：

1. `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` 或 `CLAUDE_CODE_FORK_SUBAGENT=0` —— 连功能一起关掉。能止血，但你买这个功能就是要用它。
2. 一个约 80 行的 Bun preload 脚本。它包住全局的 `setTimeout` 和 `setInterval`，用 `Function.prototype.toString` 检查每个定时器回调的源码里有没有锚点字符串（`[AgentSummary] Timer fired`），有就返回一个哑句柄而不是真正装弹。

第 2 条方案里有个细节我特别喜欢。哑定时器的延迟被设成**正好 2³¹−1 毫秒**，而不是更大。因为 Node 和 Bun 会把超过这个值的延迟当成 1 毫秒处理——你以为设了个"永不触发"的定时器，它会立刻触发。

还有为什么非得去 hook 定时器：这个模块是预编译的 Bun 字节码，改二进制里的 JS 文本没有任何效果；API 客户端也没走能被包住的 `globalThis.fetch`。能读到的只剩回调的源码字符串，所以字符串匹配成了唯一入口。

代价是它**失败即放行（fail-open）**。任何一次重命名、一个 `fn.bind` 包一层、或者换成别的定时器 API，这个钩子就什么都匹配不到，摘要会**静默回来**。脚本作者自己在文档里写：每次升级都要重新验证。

一个用户为了解决"一行状态提示太贵"的问题，需要靠匹配混淆代码里的日志字符串来打补丁，并且每次升级都要重新确认。这个方案作者自己都说了，可能和服务条款冲突。

---

## 五、Anthropic 的回应，以及两个数字为什么可以同时为真

帖子发出约 80 分钟后，一位自称 Claude Code 团队成员的工程师 Edwin Arbus 在 HN 上回复了。

他没有否认机制，否认的是**代表性**：

> 只有正在工作的后台智能体才会收到摘要调用，空闲的不会。
> 上周，在我们发送遥测的用户中，`agent_summary` 按 API 标价计算约占总成本的 **2%**，占请求量略低于 4%。在终端里（几乎所有这类调用都发生在那里），约占成本的 **6%**。

关于 16.4%，他的说法是"听起来像是一个主要跑后台智能体的团队，不像典型的会话"。

然后他说了一句我觉得很关键的话：

> That said, we agree that's a lot for a status line.

他还说，最贵的一类情况——摘要请求没命中提示缓存——已经在 **2.1.311 及之后**修掉了；团队正在做一个"便宜得多的模式：发送简短的笔记给更小的模型，而不是整段 transcript"。

现在说我的看法。

**我信那个 2%，但我认为它是看错了的数字。**

全站均值里包含了所有从来不打开后台智能体的人。这些人根本不产生摘要调用，他们会把任何百分比稀释掉。而真正在搜"Claude Code usage limit reached"的人，是同时跑好几个智能体的重度用户——**钱主要是他们在付。**

同一条讨论里，一位 20x Max 订阅者让 Opus 根据他自己的 transcript 估算：一周 API 等价花费 1,970 美元，上面还压着约 380 美元的状态行调用，占真实总额的 16%。另一位测下来"最多 8%"。还有人对 30 天内 129 场后台智能体会话做了统计：`agent_summary` 约占请求量 14%、估算费用 10%，而一场 40 个智能体的会话里达到 24%。

**2% 和 16% 不矛盾，它们在描述两个不同的分布。** 问题是其中只有一个会决定你下午三点会不会撞上限额。

---

## 六、真正让我不安的不是 16.4%

先说我不反对的部分。

摘要这个功能本身不是坏主意。多个智能体并行跑的时候，人确实需要知道每个在干嘛，而"不读 transcript 就能知道"是有价值的产品能力。Arbus 说这些摘要同时给用户和 Claude Code 自己用来追踪状态，这个理由我接受。

我反对的是三件事叠在一起：**默认打开、无法关闭、不可见。**

三件事单独看都还能辩护。合在一起，一个产品选择就变成了一笔用户不知道自己在付的账。而且这笔账不会出现在任何一张收据上——不在 session 日志里，不在计费工具的统计里，也不在文档里。它是靠有人往 OpenTelemetry 导出里看了一眼才被发现的。

但更根本的东西在这儿：

> 我们正在把"系统向人解释自己在干什么"这件事的成本，从一次性的 UI 渲染，变成每次刷新都要跑一遍模型推理。

以前状态栏显示 git 分支，那是一次纳秒级的字符串拼接。现在显示智能体在干什么，那是一次几十万 token、几百毫秒、而且**随上下文长度增长**的模型调用。

在一个传统系统里，可观测性是最便宜的那一层。日志、指标、追踪，它们的成本相对被观测的工作负载可以忽略不计——正因为便宜，你才敢全量打开，才敢默认打开。

现在这个前提反过来了。**当"让 AI 解释自己在干什么"也由 AI 来完成，可观测性就从常数成本变成了随任务规模增长的成本。** 而且它增长得比主任务更快——主任务是线性读上下文，摘要是反复读。

这才是这件事真正的分量。不是 Anthropic 收了 16% 或者 2% 的钱，而是**我们正在往一类系统里嵌入一个正反馈：越需要观测的地方，观测越贵；跑得越久，越贵；而跑得久正是我们用它的原因。**

修法很简单，简单到有点荒谬：

把它变成一个设置项。默认用便宜模式。写进文档。

Arbus 已经说了便宜模式在做。但 2.1.310 里其实已经带着一个了——发约 2000 token 的笔记，而不是整段 transcript，还可以选择发给更小的模型。**它现在是关着的，而且只有服务端 flag 能打开。**

便宜的版本早就写好了。它只是没被打开。

---

### 参考与延伸

- Flamerix AI（2026-10-29）：*Claude Code regularly sends each background agent's transcript to its model*，GitHub Gist — <https://gist.github.com/flamerix-ai/c3905705cc4205e5de3011fa2e74efce>
- Hacker News 讨论帖（2026-10-29，数百条评论，含 Edwin Arbus 的官方回复）：<https://news.ycombinator.com/item?id=50284699>
- GitHub issue #86879（rflcrz，2026-08-15 提出，要求提供官方关闭方式）：<https://github.com/anthropics/claude-code/issues/86879>
- GitHub issue #104358（摘要请求不写入会话日志）
- Claude Code 文档：Status line — <https://code.claude.com/docs/en/statusline>
- Claude Code Agent SDK 参考（TypeScript，`agentProgressSummaries` 选项）：<https://docs.claude.com/en/api/agent-sdk/typescript>
- Claude Code CHANGELOG v2.1.311：<https://dev.classmethod.jp/en/articles/20261027-cc-updates-v2-1-311>

> 说明：文中 16.4%、19.7%、96%、约 9000:1 等数据来自 Flamerix AI 自行导出的 OpenTelemetry 遥测，且"费用"指 Claude Code 客户端的逐请求 `cost_usd` 估算值，而非发票；在 Pro/Max 订阅下实际消耗的是速率限额额度，不是现金。18–26% 区间来自 issue #86879 汇总的四份独立用户自测（rflcrz 18.0% 加权 token、sylvain-artois 26% 缓存读取 token），同样为自报数据。2% / 4% / 6% 来自 Anthropic 工程师 Edwin Arbus 在 HN 的回复，统计口径为"上周发送遥测的 Claude Code 用户"，且按 API 标价计算。机制细节（`querySource: "agent_summary"`、`forkLabel: "agent_summary"`、`maxTurns: 1`、`skipTranscript: true`、锚点字符串 `[AgentSummary] Timer fired`）为本文在本机 Claude Code 2.1.308 二进制中直接 grep 验证，非二手转述。文中提及的 Bun preload 脚本为非官方方案，作者已声明其依赖内部实现、可能随版本失效、并可能与服务条款冲突，本文引用它仅用于说明"用户缺乏官方开关"这一事实，不构成使用建议。
