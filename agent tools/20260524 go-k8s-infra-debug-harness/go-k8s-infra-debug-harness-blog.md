# 一次 Go/Kubernetes 项目排障经历，让我做了一套 AI Agent 工程化约束工具

最近我在公司用 AI Agent 辅助排查一个 Go 项目中的问题。

这个项目和 Kubernetes、容器网络代理、负载均衡、failover 等基础设施场景有关，代码本身并不简单，运行时行为还依赖真实环境中的配置、日志、部署参数和旧版本依赖。

一开始，我只是想让 AI Agent 帮我更快地理解代码、定位问题、给出修复方案。但实际使用过程中，我逐渐发现：AI Agent 很强，但如果没有工程化约束，它也很容易把我们带偏。

于是，我开始和 ChatGPT 一起，把这次排障过程中的问题、教训和改进方法沉淀成了一套工具：

`go-k8s-infra-debug-harness`

它不是一个业务框架，也不是一个代码生成器，而是一套面向 Go/Kubernetes 基础设施项目的 AI Agent 工程化约束框架。

它主要用于：

- 疑难问题排查
- 证据驱动诊断
- 规格先行实现
- 编译与测试验证
- 独立代码审查
- Review 争议裁决
- 长上下文任务交接
- 团队知识资产沉淀

## 背景：一次 AI 辅助修复中的真实困境

这次问题发生在一个 Go/Kubernetes 相关项目中。

我使用的是 Claude Code 接入公司内部模型来辅助排查和修复问题。刚开始体验还不错，Agent 会主动探索代码，也能使用 subagent 去减轻主上下文压力。

但随着问题推进，我遇到了几个非常典型的问题。

### 1. 上下文越来越长，后期经常超出 context window

疑难排障不是一两轮问答能解决的。

Agent 需要读代码、看调用链、分析日志、写文档、输出方案、修改代码、再验证。到了后期，上下文会越来越长，内部模型经常因为超出 context window 报错，导致一次输出直接浪费。

我只能回滚、手动 compact、重新提问。

这带来的问题是：前面已经分析过的代码、Agent 自己输出过的文档、已经形成的判断，很容易在 compact 后丢失。

### 2. 已经读过的代码被反复重新读取

因为上下文被压缩，Agent 会忘记前面探索过什么。

于是它不得不重新读代码、重新确认路径、重新理解调用链。对于复杂项目来说，这个成本非常高。

而且更麻烦的是，Agent 不只是忘记我说过的话，也可能忘记它自己刚刚生成过的文档。

这让我意识到：不能把聊天历史当作唯一上下文。重要结论必须沉淀到文件里。

### 3. Agent 会在证据不足时猜根因

这次最关键的问题是：Agent 曾经给出过一个错误的根因猜测。

它看起来分析得很有道理，但其实缺少真实运行环境的关键证据。后来我发现它输出中提到了一个核心配置文件目录，于是我自己到环境上把配置取下来检查，才发现它之前的方向是错的。

换句话说，这个错误本可以避免。

如果 Agent 在给出猜测性原因之前，先明确要求我提供：

- 真实运行日志
- 真实配置文件
- 部署 manifest
- 运行参数
- 版本信息
- 关键命令输出

它就不应该过早下结论。

这让我意识到：疑难问题必须先建立“证据矩阵”，不能裸猜。

### 4. Subagent 用得不够彻底

Claude Code 在探索代码阶段确实使用了 explore subagent，这很有帮助。

但后续生成代码、生成调用链文档、输出实施方案文档时，很多工作又回到了主 Agent 上，导致主上下文继续膨胀。

这让我意识到：不能让主 Agent 什么都干。应该给不同 Agent 明确分工。

### 5. 我更需要文档化的知识资产，而不只是代码修改

对于简单问题，我可以接受 Agent 直接改代码。

但对于不熟悉的复杂功能，尤其是新同事也要理解的老代码，我更希望 Agent 先输出：

- 现有功能调用链文档
- 源文件、方法、关键代码块说明
- 根因分析文档
- 多种修复方案横向比较
- 最终实现规格文档

只有当这些内容被我确认后，再进入实现阶段。

这样做不仅能降低误修风险，也能把一次排障变成团队知识资产。

### 6. Agent 生成的代码可能连编译都过不了

这次还有一个很现实的问题：Agent 生成的 Go 代码最初无法编译。

原因是项目使用的 Kubernetes 依赖版本比较旧，有些 API 或方法在当前依赖中并不存在。

这说明 Agent 在实现代码前没有检查项目真实依赖版本。

所以我认为：对于 Go/Kubernetes 项目，Agent 在使用 API 前必须做依赖版本门禁，至少要检查：

- `go.mod`
- `go list -m all`
- `vendor` 目录
- 当前仓库已有用法
- 本地 module cache

不能默认使用最新 Kubernetes API。

### 7. 修复代码后，规格文档也必须同步

当我指出编译问题后，Agent 修复了代码。

但这也暴露了另一个问题：如果代码修复意味着原来的实现规格是错的，那么 Agent 不应该只改代码，而应该提醒我：是否允许同步修改规格文档。

否则代码和文档会逐渐不一致。

### 8. Review Agent 和 Implementer Agent 需要对峙机制

代码提交后，我用另一个 Agent 对新增代码做 review，发现了几个问题。

这让我想到一个更工程化的流程：

- Implementer Agent 负责按规格实现
- Reviewer Agent 负责独立审查
- 如果双方意见不一致，需要 Dispute Arbiter Agent 判断

裁决时要明确：

- 哪些问题是真实有效的
- 哪些是误报
- 哪些部分有效
- 哪些需要人类决策

这比“一个 Agent 自己写、自己说没问题”要可靠得多。

## 工具目标：给 AI Agent 套上一套工程化安全带

基于这些问题，我开始沉淀这套工具。

它的核心思想是：

> 不要让一个主 Agent 什么都做。  
> 要把探索、诊断、规格、实现、验证、审查、争议裁决拆开。  
> 每一步都有输入、输出、边界和人类审批门禁。

这套工具最终形成了一个子目录：

```text
go-k8s-infra-debug-harness/
```

它面向的主要场景是：

- Go 项目
- Kubernetes 项目
- 容器网络
- 网络代理
- CNI
- 负载均衡
- failover / keepalived
- controller / informer
- 企业内部旧依赖版本项目
- 疑难排障
- 规格化修复

它支持三类 AI Coding Agent 工具：

```text
Codex
Claude Code
OpenCode
```

对应目录包括：

```text
.codex/agents/
.claude/agents/
.opencode/agents/
```

同时也抽象了一套工具无关的角色契约：

```text
agents-spec/
```

以及可复用技能：

```text
.agents/skills/
```

## 核心措施一：Evidence Matrix，禁止裸猜根因

这套工具中最重要的 skill 之一是：

```text
evidence-matrix
```

它的作用是强制 Agent 把内容分成：

- 已知事实
- 可用证据
- 缺失证据
- 暂时假设
- 候选假设
- 反证
- 当前最佳解释
- 最终根因

尤其重要的是：不能在证据不足时直接给最终根因。

每个假设都必须包含：

- 支持证据
- 反向证据
- 缺失证据
- 置信度
- 下一步验证方式

置信度分为：

```text
Low
Medium
High
Confirmed
```

只有当源码证据、运行时证据、复现或验证结果都能对齐时，才能标记为 `Confirmed`。

这解决了我这次遇到的最大问题：Agent 过早猜根因。

## 核心措施二：Code Call Chain，先理解现有调用链

对于复杂老项目，尤其是新同事不熟悉代码时，直接让 Agent 改代码是很危险的。

所以我沉淀了：

```text
code-call-chain
```

它要求 Agent 先做只读代码探索，并输出调用链文档。

内容包括：

- 入口点
- 核心文件
- 核心结构体
- 核心方法
- 配置加载流程
- 运行参数构造
- 外部交互
- 错误处理
- 日志点
- goroutine / informer / watcher / controller 生命周期
- 缓存和状态流转
- 版本敏感 API

这样做的好处是：在动手修复前，先把“系统现在是怎么工作的”讲清楚。

## 核心措施三：Go/K8s Dependency Gate，避免使用不存在的 API

这次 Go 代码编译失败给了我一个很强的教训。

AI Agent 很容易按照“它知道的新版本 API”写代码，但企业项目未必使用新版本依赖。

所以工具里加入了：

```text
go-k8s-dependency-gate
```

它要求在涉及 Go/Kubernetes 依赖敏感 API 时，先检查当前项目真实版本。

辅助脚本包括：

```text
scripts/agent/check-go-k8s-deps.sh
```

它可以帮助检查某个 API、类型、字段、方法是否存在于当前依赖环境中。

这类检查对 Kubernetes 项目非常重要。因为很多项目使用较旧版本的 `client-go`、`apimachinery`、`controller-runtime`，不能假设最新 API 可用。

## 核心措施四：Implementation Spec，复杂修改必须先写规格

对于复杂修改，这套工具要求先写：

```text
05-implementation-spec.md
```

也就是实现规格。

实现规格要包含：

- 背景
- 目标和非目标
- 受影响文件和方法
- 当前行为
- 目标行为
- 详细设计
- 依赖和兼容性约束
- 测试计划
- 验证命令
- 回滚方案
- 风险和缓解措施
- 验收标准

只有规格经过人类确认后，Implementer Agent 才能动手写代码。

如果实现过程中发现规格错误、不完整、不安全、和依赖版本冲突，Agent 必须停止，并输出：

```text
Spec Deviation Notice
```

也就是规格偏离通知。

这能避免一种很常见的问题：Agent 发现原方案不对，但悄悄改了方向，最后代码和文档都对不上。

## 核心措施五：Verification Gate，不验证不算完成

这套工具要求：实现完成不等于任务完成。

必须经过验证。

对应 skill 是：

```text
verification-gate
```

验证报告需要记录：

- 执行了哪些命令
- 哪些通过
- 哪些失败
- 哪些跳过
- 哪些环境不可用
- 关键失败输出
- 是否和当前修改相关
- 剩余风险

对于 Go 项目，辅助脚本包括：

```text
scripts/agent/verify-go.sh
```

可以执行类似：

```bash
scripts/agent/verify-go.sh ./...
scripts/agent/verify-go.sh ./pkg/xxx/...
```

关键原则是：

> 没有命令输出，就不能声称验证通过。

## 核心措施六：Review Dispute，让审查和实现分离

我不希望同一个 Agent 既写代码，又自己宣布 review 通过。

所以工具里设计了独立角色：

```text
reviewer
```

Reviewer 的任务是对 final diff 做证据化审查，输入包括：

- approved implementation spec
- final diff
- verification report
- dependency report
- code call-chain document
- project rules

每个 issue 必须包含：

- 严重级别
- 类别
- 置信度
- 证据
- 风险
- 建议修复方式
- 是否影响规格
- 是否需要重新验证

如果 Reviewer 和 Implementer 对某个 issue 有分歧，则交给：

```text
dispute-arbiter
```

它会把争议分类为：

- Valid Issue
- False Positive
- Partially Valid
- Needs Human Decision

这能把“Agent 之间互相说服不了”的问题变成一个可审查、可记录、可决策的流程。

## 核心措施七：Context Handoff，防止长上下文丢失

长任务一定会遇到 context 变长、compact、切换工具、换会话的问题。

所以工具里有：

```text
context-handoff
```

它要求在关键节点写入：

```text
docs/agent/cases/<case-id>/context-handoff.md
```

内容包括：

- 当前目标
- 当前阶段
- 已确认事实
- 暂时假设
- 已做决策
- 已产出文档
- 已检查文件
- 已执行命令
- 当前假设
- 阻塞点
- 下一步建议
- 尚未允许的动作
- 需要人类审批的事项
- 恢复提示词

这解决了一个非常实际的问题：不能让聊天历史成为唯一上下文。

## 关键 Subagents 角色

这套工具中，核心角色大致如下。

### orchestrator

主控编排 Agent。

它不应该什么都自己做，而是负责：

- 判断任务复杂度
- 选择工作流
- 调度其他 Agent
- 管理人类审批门禁
- 维护任务状态
- 决定什么时候需要 handoff

### code-explorer

只读代码探索 Agent。

负责读代码、梳理调用链、输出调用链文档，不做最终根因判断，不实现代码。

### evidence-diagnoser

证据驱动诊断 Agent。

负责构建证据矩阵、假设矩阵，主动要求用户提供日志、配置、部署信息、命令输出等运行时证据。

### spec-writer

规格编写 Agent。

负责把已确认的分析、调用链、根因和方案转换为实现规格。实现前必须要求人类审批。

### implementer

实现 Agent。

只按已批准的 implementation spec 写代码。不能擅自扩范围，不能改方案，发现规格错误必须停下来。

### verifier

验证 Agent。

负责跑编译、测试、静态检查、依赖验证，并输出 verification report。

### reviewer

审查 Agent。

负责对 final diff 进行证据化审查，不修改代码，不用个人偏好冒充 blocker。

### dispute-arbiter

争议裁决 Agent。

当 Reviewer 和 Implementer 有分歧时，它基于证据判断 issue 是否有效、误报、部分有效或需要人类决策。

## 推荐工作流

这套工具把任务分成三类。

### Simple Fix

适合小改动。

流程大致是：

```text
orchestrator
-> implementer
-> verifier
```

### Medium Change

适合影响模块行为、测试或文档的中等改动。

流程大致是：

```text
orchestrator
-> code-explorer
-> spec-writer
-> implementer
-> verifier
-> reviewer
```

### Difficult Debugging

适合生产/测试环境问题、Kubernetes、网络、failover、并发、旧依赖版本、陌生代码等高风险问题。

流程大致是：

```text
orchestrator
-> evidence-diagnoser
-> code-explorer
-> evidence-diagnoser
-> spec-writer
-> human approval
-> implementer
-> verifier
-> reviewer
-> dispute-arbiter when needed
-> doc-code-consistency
-> context-handoff
```

这也是我最看重的场景。

## 如何使用

如果使用 Codex，可以参考：

```text
go-k8s-infra-debug-harness/docs/agent/codex-usage.md
```

如果使用 Claude Code，可以参考：

```text
go-k8s-infra-debug-harness/docs/agent/claude-code-usage.md
```

如果使用 OpenCode，可以参考：

```text
go-k8s-infra-debug-harness/docs/agent/opencode-usage.md
```

如果要接入真实项目，可以参考：

```text
go-k8s-infra-debug-harness/docs/agent/adoption-guide.md
```

如果团队主要使用中文交流，可以直接使用中文版工具包：

```text
go-k8s-infra-debug-harness-zh-CN/README.md
go-k8s-infra-debug-harness-zh-CN/docs/agent/codex-usage.md
go-k8s-infra-debug-harness-zh-CN/docs/agent/claude-code-usage.md
go-k8s-infra-debug-harness-zh-CN/docs/agent/opencode-usage.md
go-k8s-infra-debug-harness-zh-CN/docs/agent/adoption-guide.md
```

中文版工具包的目录结构和功能设计与英文版保持一致，只是把面向人类和 Agent 的 Markdown 说明、角色契约、skills 文档、usage guide 等内容翻译成中文，方便团队内部协作、评审和知识沉淀。

一个复杂排障任务的起始 prompt 可以是：

```text
Use the orchestrator workflow.

Classify this task as Difficult Debugging.

Problem:
<describe the observed problem>

Project context:
This is a Go/Kubernetes container networking or proxy-related project.
Runtime behavior may depend on deployment manifests, generated configuration, Kubernetes resources, node networking state, logs, and dependency versions.

Rules:
- Do not modify code yet.
- Do not claim a final root cause without evidence.
- First identify what runtime evidence is needed.
- Use code exploration to produce source-backed call-chain documentation.
- Use evidence-matrix diagnosis before proposing a fix.
- Ask me for missing logs, configuration directories, manifests, versions, or command output.
- Check Go/Kubernetes dependency versions before proposing APIs.
- Produce an implementation specification before coding.
- Ask for my approval before implementation.
- Verify after implementation.
- Review the final diff against the approved specification.
- Use context-handoff if the task becomes long-running.
```

如果是 OpenCode，则可以用：

```text
@orchestrator

Classify this task as Difficult Debugging.

Problem:
<describe the observed problem>

Rules:
- Do not modify code yet.
- Do not claim a final root cause without evidence.
- Use @evidence-diagnoser to identify required evidence.
- Use @code-explorer to produce call-chain documentation.
- Use @spec-writer before implementation.
- Ask for my approval before implementation.
- Use @implementer only after approval.
- Use @verifier after implementation.
- Use @reviewer after verification.
- Use @dispute-arbiter if reviewer and implementer disagree.
```

如果使用中文版工具包，也可以直接使用中文提示词，例如：

```text
使用 orchestrator 工作流。

请将这个任务分类为：疑难排障。

问题现象：
<描述观察到的问题>

项目背景：
这是一个 Go / Kubernetes 容器网络或代理相关项目。
运行时行为可能依赖部署清单、生成配置、Kubernetes 资源、节点网络状态、运行日志和依赖版本。

规则：
- 现在不要修改代码。
- 在证据不足时，不要声明最终根因。
- 首先识别需要哪些运行环境证据。
- 使用 code-explorer 输出基于源码证据的调用链文档。
- 使用 evidence-diagnoser 构建证据矩阵和假设矩阵。
- 如果需要运行日志、真实配置目录、部署清单、依赖版本或关键命令输出，请明确向我索要。
- 在提出实现方案前，先检查 Go / Kubernetes 依赖版本。
- 在编码前，先使用 spec-writer 输出实现规格。
- 实现前必须请求我的审批。
- 只有在我审批后，才能使用 implementer 修改代码。
- 实现完成后，必须使用 verifier 做编译、测试或静态检查验证。
- 验证后，使用 reviewer 对最终 diff 做独立审查。
- 如果 reviewer 和 implementer 对某些 issue 有争议，使用 dispute-arbiter 做争议裁决。
- 如果任务变长、需要 compact 或切换工具，使用 context-handoff 生成上下文交接文档。
```

如果是 OpenCode 中文场景，可以写成：

```text
@orchestrator

请将这个任务分类为：疑难排障。

问题现象：
<描述观察到的问题>

规则：
- 现在不要修改代码。
- 不要在证据不足时猜测最终根因。
- 使用 @evidence-diagnoser 识别需要补充的运行环境证据。
- 使用 @code-explorer 输出调用链文档。
- 使用 @evidence-diagnoser 构建证据矩阵。
- 如果需要日志、配置、部署清单、版本信息或命令输出，请明确向我索要。
- 在实现前使用 @spec-writer 编写实现规格。
- 实现前必须请求我的审批。
- 审批后才能使用 @implementer。
- 实现后使用 @verifier 验证。
- 验证后使用 @reviewer 审查。
- 如果 reviewer 和 implementer 有争议，使用 @dispute-arbiter 裁决。
```

## 我们是怎么一步步搭出来的

这套工具不是一开始就设计完整的。

它是从一次真实排障体验开始的。

最初只是觉得：AI Agent 在疑难问题上很容易上下文爆掉、重复读代码、忘记前面分析、猜根因、生成无法编译的代码。

然后我和 ChatGPT 一起，把这些问题逐个拆开：

- 如何防止裸猜根因？
- 如何让 Agent 主动索要日志和配置？
- 如何把调用链文档化？
- 如何让规格先于代码？
- 如何检查 Go/K8s 依赖版本？
- 如何强制验证？
- 如何让 Reviewer 和 Implementer 分离？
- 如何处理 review 争议？
- 如何在 compact 前保存上下文？
- 如何适配 Codex、Claude Code、OpenCode？
- 如何让这个工具变成可复制到真实项目里的 harness？

最后逐步形成了：

```text
.agents/skills/
agents-spec/
.codex/agents/
.claude/agents/
.opencode/agents/
docs/agent/
scripts/agent/
```

并且在英文版基础上，又生成了一套完整可用的中文版本，方便中文团队直接阅读、讨论和使用。

然后又把它收拢到一个更明确的子目录：

```text
go-k8s-infra-debug-harness/
```

后来考虑到团队主要使用中文交流，我又把整套工具翻译并整理成了中文版工具包：

```text
go-k8s-infra-debug-harness-zh-CN/
```

这样英文版和中文版可以并存：

```text
go-k8s-infra-debug-harness/          # 英文版
go-k8s-infra-debug-harness-zh-CN/    # 中文版
```

这个过程本身也很有意思：我们不是只让 AI 生成业务代码，而是反过来用 AI 协助设计一套“约束 AI 的工程体系”。

这有点像给 AI Agent 系上一根安全绳。

## 我的理解

经过这次实践，我越来越觉得：

> AI Agent 不是不能用于复杂工程问题。  
> 但不能裸用。  
> 它需要流程、边界、证据、规格、验证和审查。

如果只是简单问一句“帮我修这个 bug”，Agent 很可能会给出一个看起来合理但缺少证据的答案。

但如果我们把任务拆成：

```text
先读代码
再要证据
再做诊断
再写规格
再审批
再实现
再验证
再审查
再沉淀
```

AI Agent 的可控性和产出质量就会明显提高。

这套 `go-k8s-infra-debug-harness` 目前还只是一个初始版本，但它已经把我这次排障中踩到的坑转化成了可复用规则。

它不追求让 Agent 更“自由”，而是让 Agent 更“可靠”。

这也是我觉得它值得沉淀和推广的原因。

## 后续计划

后续我希望继续做几件事：

1. 在真实 Go/Kubernetes 项目里跑一个完整 case。
2. 沉淀一个脱敏示例案例，方便团队学习。
3. 把最终一致性检查脚本化。
4. 增加安装或同步脚本，方便复制到真实项目。
5. 继续完善中文版工具包和示例，让中文团队更容易直接落地。
6. 根据真实使用反馈继续优化 subagents 和 skills。

如果这套方法能让一次复杂排障少走一点弯路，少一次错误猜测，少一次无法编译的修复，少一次上下文丢失，那它就已经很有价值了。
