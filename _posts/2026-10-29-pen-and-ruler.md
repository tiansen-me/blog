---
title: 77% 与 516 个补丁：当写代码的和验收代码的是同一个东西
date: 2026-10-29 20:10:21 +0800
categories: ['engineering']
tags: ['智能体', '安卓', '工程实践', '代码审查']
slug: pen-and-ruler
image:
  src: /assets/2026-10-29-wechat-cover-2.35x1.png
  alt: 笔与尺子：一支钢笔的笔尖化入刻着刻度的横杆
---

## 两个窗口

如果你今年用 Claude Code 或者 Codex 写 Android，你的桌面大概率是这样：左边一个终端跑 agent，右边开着一个 Android Studio。右边的窗口很多时候连代码都不写，你留着它只为三样东西——模拟器、Logcat、Compose Preview。

然后你要在两个窗口之间来回搬上下文：把报错贴过去，把 patch 搬回来，重新 build 一次，看它有没有跑起来。

2026 年 10 月 28 日，Google 在 Android Dev Summit 上把这个割裂拆掉了，方式是让出自己的位置。

Android Studio Rabbit 2 上线了 Bring Your Own Agent。`Settings > Tools > AI > Agents` 里有一个 marketplace，可以装超过 40 个兼容 ACP 的 coding agent——官方点名的是 Anthropic 的 Claude Agent、OpenAI 的 Codex、Google 自家的 Antigravity，还有 Junie。可以装好几个，同时在 IDE 里跑。装进去之后，这些 agent 直接继承 IDE 对项目的理解：项目结构和构建配置、符号解析、Compose Preview 渲染、模拟器、Logcat。

底下那层叫 Agent Client Protocol（ACP），一个开放协议。按它的规范，本地 agent 以编辑器子进程的身份运行，双方用 JSON-RPC over stdio 通信。这条很重要，它解释了一件表面上看很奇怪的事：**一次集成为什么能覆盖四十多个 agent**。Android Studio 只需要实现一次协议，剩下的是 agent 那边的适配成本，不由 Google 承担。

大多数报道写到这儿就收尾了，标题通常是「Google 开放生态」之类的。我觉得这条新闻真正有意思的部分在另一个地方，藏在几条不太起眼的 CLI 命令里。

## Google 放手的，是它本来也锁不住的东西

先把商业逻辑说清楚，因为它能解释后面所有的技术选择。

平台厂商的传统剧本是 bundling：我的 IDE、我的云、我的模型。Google 这次反着来。Matthew McCullough（Android 产品副总裁）给的公开说法是给开发者选择模型和 agent 的自由；Adarsh Fernando（Android 开发者工具高级产品经理）在 10 月 28 日的官方博客里，把目标描述成给开发者和他们的 agent「造高质量 Android 应用所需的工具和知识」。

这些话我不怀疑，但也不够。算一笔账就明白了：ACP 规定本地 agent 是编辑器的子进程，通信走 stdio。协议一旦成立，Google 在技术上就没有阻断点——你没法阻止用户在自己机器上的子进程里跑别人的模型，除非把 IDE 收回去。既然拦不住，不如把它做成人人可用的入口，至少入口还是自己的。

那能锁什么？往上走一层：**数据**。

这次发布里有两组东西属于这一类。一组是 IDE 提供的项目上下文（符号、依赖图、Compose 预览），另一组是只有 Google 拿得出的生产数据：Android vitals、Firebase Crashlytics、以及一个可以远程 provision 真机的设备实验室。

所以准确的说法不是「Google 开放了」，是**锁的位置从模型迁移到了数据源**。你在 Android Studio 里装哪个模型，Google 真的不在乎；在乎的是你的崩溃堆栈、ANR 和帧率数据从哪条管道流出去。这是一笔比卖 Gemini 订阅更划算的交易。

## 被拆掉的那道 friction

我对 coding agent 过去两年的印象，可以用一个分工来概括：**agent 负责写，人负责验收。**

这个分工里有一种很朴素的安全感，它的来源其实不是「人更聪明」，而是 friction。agent 想证明自己写对了，得走完编译、起模拟器、复现路径、看日志、抓堆转储这一串动作，而这些动作过去大多需要一个人在 IDE 里操作。人不一定每次都认真，可能只是扫一眼。但这道 friction 客观上让「生成」和「判定」发生在两个不同的地方——**而这件事本身就是一层安全设施**，只是从来没有人把它写进文档。

这次发布拆的就是它。四件小事，放在一起看会很清楚：

**`android preview`。** 不用起模拟器、不用开 IDE，Compose Preview 直接在终端渲染。Google 内部测试里，agent 用它做 UI 任务快了 77%，token 最多降到三分之一。

**LeakCanary 集成。** 开源的内存泄漏检测库被塞进了 CLI 和 Android Studio Profiler，关键是堆转储在开发机上分析，不在手机上跑。这个细节有第二层意思：在被测设备上跑分析会让被测 app 变慢，测出来的数据因此不可信，而把不可信的结果喂回去让 agent 再猜一轮，成本是 token。官方演示里，Claude Code 接到一句指令就在多台设备上并行跑关键用户路径、读取堆转储、把泄漏追到具体的那几行代码、改完再跑一遍验证。Google 用 Claude Opus 5.5 做的 A/B：定位并修复内存泄漏快近一倍，token 少约 25%。

**`android signals`。** Android vitals 和 Firebase Crashlytics 的生产数据进同一个终端。线上崩溃、ANR、内存问题现在变成 agent 能直接查询的东西。

**`android device remote`。** 从 Google 的真机实验室给 agent 调一台物理设备，用来追那些只在特定机型上复现的 bug。

把这四条摆在一起：写代码的那东西现在能看见自己做的 UI，能运行检测自己代码的工具，能读生产环境的崩溃堆栈，还能拿到真机。**闭环合上了。**

我不想把它说成一件坏事。上面每一步单独拿出来都是实打实的效率提升，而且提升的性质相当健康——这个我第三节再说。但它确实换掉了那个默认假设：过去「生成」和「判定」分离是免费的、由物理距离保证的；**现在它变成一个需要有人主动维持的选项。**

然后你会注意到，Google 在同一个发布里配了两件对冲工具：

- **Change Review**：从 agent 的待提交改动或者 Git 变更窗口起一个 review session，Android Studio 生成一份潜在问题报告；在 agent 工具窗口里也能用 `/review` 命令触发。
- **Android Bench 2.0**：评测不同模型和 agent 组合在天级任务上的表现，不只看功能有没有做出来，也把**完成时间和花掉的钱**算进分数。

一家公司同时发布「让 agent 自我验证的工具」和「验证 agent 的工具」，这本身就是一个表态。它知道自己打开的是什么。

## 77% 不是智力，是省掉的无用功

这四个数字——77%、三倍 token、近一倍速度、25% token——很容易被读成「模型又变强了」。看一眼各自的机制会发现它们是同一类东西。

`android preview` 省的不是思考时间，是**等模拟器起来**的时间，以及等待过程中为了搞清楚到底哪里不对而反复试错的 token。把渲染结果直接放回终端，本质是给 agent 修了一条反馈周期极短的回路。

LeakCanary 那条更直白，省掉的是 garbage-in-garbage-out 那一轮循环。

30 多条 Android Skills（覆盖 20 多个领域）解决的是另一类无用功：模型没训过新 API，就会凭印象编一个不存在的调用，然后一轮一轮地试错。Google 的解法是直接喂文档——提审前检查 Play 政策、权限安全审查、折叠屏和大屏适配。这一步把「让它知道」从「等下一个版本的训练」改成了「现在给」。

顺着这条路，Lightbuild 就很好懂了：一个基于 YAML schema 的声明式构建系统，人和 agent 都能读能改，目前还在测试阶段。构建配置一直是 agent 最容易抓瞎的地方——Gradle DSL 是给 JVM 工程师读的，不是给语言模型读的。把它换成 YAML，等于把最后一块「只有人看得懂」的地方也拆了。

我的结论是：**2026 年 agent 性能的瓶颈是上下文成本，不是智力。** Google 这场发布卖的不是更聪明的模型——它连自己的 Gemini 都没推荐——而是「少做无用功」。

## 29,000 与 516

现在泼冷水。

先看 Anthropic 那组数。它的 OSS Scanner 在六个月里扫了一批世界上使用最广的开源项目，Claude 提出**超过 29,000 个**候选漏洞。复核流水线靠六家外部安全研究公司做复现和分诊，到 10 月 2 日只过完了约 6,000 个。

漏斗的形状是这样的：

- 已审阅 **6,123** 个，外部公司确认有效 **5,674** 个——**准确率 92.7%**，相当漂亮；
- 发出了 6,157 份报告，拿到 584 个 CVE / GHSA 编号；
- 真正在上游**修好的：516 个**；
- 还有约 **23,000** 个候选没审。

从「确认」到「修好」这一级保留了 9%。注意坏的是哪一半：**不是「找」，是「修」，而这恰好是这次 Android 发布试图自动化的那一半。**

第二个例子更锋利，也更公平。10 月 13 日，Linus Torvalds 提交了一组补丁，重做 Linux page cache 里的 folio 引用计数瓶颈——这个问题 Matthew Wilcox 在 2024 年就从客户负载里发现了，Torvalds 自己也带着一个坏版本跑了十八个月。Torvalds 的说法是：实现、测试和初版文档由 Claude Code 完成，review 也是它在指导下做的，基准测试他自己跑。

数字很吓人。在 64 线程 Threadripper 的合成基准上，每秒读从 1,330 万次提到 **3.04 亿次**，每次读消耗的周期数从接近 20,000 降到 **735**。

然后 Kiryl Shutsemau 在 `filemap_read_rcu_batch()` 里找到一个竞态：folio 被替换之后紧接着 `i_size` 扩展，有可能把**从未被写过的数据**暴露出去。Torvalds 随后把后续开发工作交给了 Shutsemau——他此前恰恰也在同一个瓶颈上做过工作。

我特别喜欢这个例子，因为它把边界画得比以前任何争论都清楚。这不是「AI 写的代码有 bug」的老故事。真实情况是：**这套补丁在合成基准里赢得非常漂亮，问题藏在并发路径里，而写它的那个东西会跑基准，不会给并发写证明。**

收益也不是白拿的，而且 Torvalds 自己在补丁说明里写清楚了：只要在单个 CPU 上排了超过 1,024 个页面，延迟回收就得强制走一次 expedited grace period；单线程 16 字节读因为会写入未映射的用户页，多触发一次缺页，每次读从约 4,150 周期涨到 6,550 周期。他也明确说过，真实负载里的提升远没有合成基准那个数字夸张。

把两组数据并排放：

> 一套扫描流水线能提出 29,000 个候选漏洞，确认环节有 92.7% 的准确率，最后在上游修掉 516 个。
> 一套能让吞吐涨 23 倍的补丁，被人从里面拎出一个竞态。

这不是在说这些技术不行。是在说：**生成能力和验收能力之间的缺口，性质变了。** 三年前缺的是产能，现在缺的是把产出变成可信结论的能力。Google 这次发布的应对，是把工具做得更强、闭环得更紧——这条路没错，但它走的方向是「给写字那个角色的能力」，而不是「给验收那个角色」。

Change Review 和 Android Bench 2.0 是两个例外：它们输出的是给人看的报告，不是给 agent 用的工具。

## 几句必须说的保留

关于这些数字，有三件 Google 没有说。

**一、性能数据全部来自 Google 自己的内部测试。** The New Stack 的原话是公司「对如何测量几乎没有说明」。LeakCanary 那条尤其单薄：**一个 A/B，一个模型**（Claude Opus 5.5）。这是一个值得去试的理由，不是对你代码库的承诺。

**二、「40+ agent」是协议兼容性，不是质量认证。** 谁都能写一个遵守 ACP 的 agent。dev.to 上 Paul Lovelace 那篇分析里有一句说得最干脆：「Protocol compatibility is not quality.」，协议兼容不等于可用。他给出的判断办法和本文结尾的建议是同一个：**拿同一个真实任务跑两个 agent，比较 diff、测试结果和 token 账单**，其余的都是宣传。

**三、大部分应用层 API 还没到 stable。** Agentic API 是 alpha、A2UI 是 alpha、AppFunctions 是 beta、Connected Apps Platform 是 preview。在这些上面做生产功能，赌的是 Google 的路线图，不是成品平台。

## 今天可以做的一件小事

最后给一件今天就能做的事，顺手也说说那些容易被忽略的安装细节。

Android CLI 现在作为官方插件装进 Claude Code、Codex 和 Antigravity。以 Claude Code 为例：

```bash
claude plugin marketplace add android/android-cli-plugin
claude plugin install android-cli@android-cli-plugin
```

Codex 那边第二个动词是 `add` 不是 `install`。插件仓库是公开的 `android/android-cli-plugin`，Apache-2.0，但它的 README 说得很清楚：**仓库里只有插件本身**，CLI 还得单独装，然后 `android update`、`android init`——`android init` 会把 android-cli skill 装上。

CI 里跑的话有一个开关值得留意：`--no-metrics` 关掉数据收集。文档说 `~/.androidrc` 可以每行一个 flag 自动生效，**但没有明说 `--no-metrics` 在里面能不能用**，所以别想当然，先在流水线里显式传。

然后是那件小事：**挑一个你已经知道答案的真实任务**——一个带着失败测试的 bug、一次 Compose 迁移、一轮权限审计——把它同时交给两个不同的 agent，比较三样东西：diff、测试结果、token 账单。

「已经知道答案」这五个字是重点。

因为当尺子交到了写字那只手里，唯一还在外面、还能用来校准它的，就是你事先知道的那个答案。它现在比以往任何时候都值钱，而它恰好只存放在一个地方——你的脑子里。没有任何一份文档会替你留着它。

Google 这次拆掉的不是一堵墙，而是「写」与「判」之间最后那层由物理距离维持的隔离。剩下的只有一个问题：**谁负责不同意。**

答案暂时还是人。但这也是第一次，人不再被夹在中间，而是被挪到了外圈。站得远一点确实看得清一点，代价是——你得自己主动站过去。

---

### 参考与延伸

- Google（2026-10-28）：*Android Dev Summit 2026 developer tools and performance*，Android Developers Blog — <https://developer.android.com/blog/posts/android-dev-summit-26-developer-tools-performance>
- Amanda Caswell（2026-10-28）：*Google just handed its rivals the keys to Android*，The New Stack — <https://thenewstack.io/android-agents-build-debug/>
- Amanda Caswell（2026-10-11）：*Claude found 29,000 possible bugs in open source. Only 516 have been fixed.*，The New Stack — <https://thenewstack.io/anthropic-oss-scanner-vulnerabilities/>
- Amanda Caswell（2026-10-14）：*Linus Torvalds let Claude Code fix the Linux page cache. Then reviewers found a race.*，The New Stack — <https://thenewstack.io/torvalds-claude-code-kernel/>
- Paul Lovelace（2026-10-29）：*Android Studio Just Opened to 40+ AI Agents — Explained Honestly*，dev.to — <https://dev.to/paullovelace/android-studio-just-opened-to-40-ai-agents-claude-codex-and-antigravity-in-one-ide-explained-28gb>
- Android CLI 插件仓库（Apache-2.0）：<https://github.com/android/android-cli-plugin>
- Agent Client Protocol 规范：<https://agentclientprotocol.com/>
- Anthropic OSS Scanner 披露仪表盘：<https://red.anthropic.com/2026/cvd/>

> 说明：文中 77%、三倍 token、近一倍速度与 25% token 四项数据均出自 Google 内部测试，其中 LeakCanary 一项为与 Claude Opus 5.5 的单个 A/B 实验，Google 未公开测量方法。Linux page cache 补丁先后存在两个版本（最初的一组补丁与后续使用 `PG_rcu_free` 标志的 `rcu-free-v3`），媒体报道对补丁数量有八与十的出入，本文采用 Threadripper 一组数字并注明为合成基准；Torvalds 本人已说明真实负载中的提升远低于该数字。Lightbuild 目前仍在测试阶段，Agentic API 与 A2UI 为 alpha、AppFunctions 为 beta、Connected Apps Platform 为 preview。
