# Java 微服务 / Go + K8s 开发人员推荐使用的 Claude Code 扩展工具清单

> 适用人群：从事 Java 微服务后端开发、Go 语言 K8s 容器化网络代理工具开发的软件开发人员。  
> 适用场景：使用 Claude Code 辅助完成代码需求开发、软件缺陷修复、代码审查、测试补充、项目上下文沉淀、K8s/微服务问题排查。

---

## 一、背景说明

对于长期从事后端开发的工程师来说，Claude Code 不只是一个“帮我写代码”的工具，更像是一个可以长期协作的智能开发助手。

尤其是在以下场景中，它的价值会非常明显：

- 接手一个复杂存量项目，需要快速理解目录结构、核心模块、调用链路；
- 开发 Java 微服务需求，例如 Spring Boot、Kafka、Elasticsearch、数据库、接口联调等；
- 修复线上或测试环境缺陷，例如异常日志、线程池、事务、消息堆积、查询慢、数据不一致等；
- 开发 Go / K8s 相关组件，例如 controller、operator、network proxy、CNI、iptables、Service、EndpointSlice 等；
- 在提交代码前进行 PR review、测试覆盖检查、安全风险检查；
- 将项目经验沉淀到 CLAUDE.md、skills、subagents、hooks 等配置中，让 Claude Code 越用越懂项目。

Claude Code 的扩展能力主要包括：

- **Plugins**：插件包，可以包含 skills、agents、hooks、MCP servers 等；
- **MCP Servers**：连接 GitHub、GitLab、K8s、数据库、文档、浏览器等外部系统；
- **Skills**：将固定工作流封装成可复用技能；
- **Subagents**：面向特定任务的专用 agent，例如代码审查、架构分析、测试分析；
- **Hooks**：在工具调用、文件编辑、提交前后等节点加自动化规则和安全护栏；
- **CLAUDE.md / Memory**：沉淀项目规则、架构说明、编码规范和经验；
- **LSP Code Intelligence**：为 Java、Go 等语言提供更接近 IDE 的代码理解能力。

---

## 二、优先推荐安装的官方 / 官方市场插件

官方插件市场 `claude-plugins-official` 可通过 Claude Code 的 `/plugin` 命令发现和安装。

常用安装方式：

```bash
/plugin install <plugin-name>@claude-plugins-official
```

### 1. `jdtls-lsp`

- **类型**：官方插件 / Java LSP
- **推荐指数**：★★★★★
- **适合场景**：Java 微服务项目必装
- **主要用途**：
  - Java 代码跳转；
  - 类型分析；
  - 引用查找；
  - 重构辅助；
  - Spring Boot / Maven / Gradle 多模块项目理解。

如果你的主力项目是 Java 后端服务，尤其是 Spring Boot 微服务，这个插件建议优先安装。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/jdtls-lsp

---

### 2. `gopls-lsp`

- **类型**：官方插件 / Go LSP
- **推荐指数**：★★★★★
- **适合场景**：Go + K8s 网络代理、controller、operator 项目
- **主要用途**：
  - Go 代码智能分析；
  - package / symbol 跳转；
  - 引用查找；
  - Go 项目重构；
  - 辅助理解 client-go、controller-runtime、K8s 相关代码。

如果你维护的是 Go 语言 K8s 容器化网络代理工具，这个插件基本属于必装。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/gopls-lsp

---

### 3. `feature-dev`

- **类型**：官方插件 / Skill + Subagents
- **推荐指数**：★★★★★
- **适合场景**：需求开发、复杂功能改造、存量代码增强
- **主要用途**：
  - 需求澄清；
  - 代码探索；
  - 架构设计；
  - 实现计划拆分；
  - 代码修改；
  - 质量审查；
  - 总结沉淀。

该插件会引入多个专用 agent，例如：

- `code-explorer`：分析现有代码结构；
- `code-architect`：设计实现方案；
- `code-reviewer`：审查实现质量。

对于“我要在已有项目里新增一个需求”的场景，它非常适合。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/feature-dev

---

### 4. `code-review`

- **类型**：官方插件 / PR Review
- **推荐指数**：★★★★★
- **适合场景**：缺陷修复后检查、PR 审查、代码提交前检查
- **主要用途**：
  - 并行启动多个审查 agent；
  - 检查明显 bug；
  - 检查是否符合 CLAUDE.md 中的项目规范；
  - 分析 git blame 历史上下文；
  - 根据置信度过滤误报。

对于缺陷修复场景，我建议每次改完之后都跑一遍。  
它就像一个嘴上不饶人但真心为你好的代码同事。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-review

---

### 5. `pr-review-toolkit`

- **类型**：官方插件 / 多个 review subagents
- **推荐指数**：★★★★★
- **适合场景**：PR 质量检查、测试覆盖分析、潜在缺陷发现
- **主要用途**：
  - 测试覆盖检查；
  - 静默失败检查；
  - 注释准确性检查；
  - 类型设计检查；
  - 通用代码审查；
  - 代码简化建议。

它比 `code-review` 更细粒度，更像一组“专项审查员”。

对于 Java 微服务和 Go / K8s 项目，尤其推荐关注：

- `pr-test-analyzer`
- `silent-failure-hunter`
- `code-simplifier`

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/pr-review-toolkit

---

### 6. `commit-commands`

- **类型**：官方插件 / Slash Commands
- **推荐指数**：★★★★☆
- **适合场景**：提交代码、创建 PR、清理无效分支
- **主要用途**：
  - `/commit`：自动生成符合项目风格的 commit message；
  - `/commit-push-pr`：提交、推送并创建 PR；
  - `/clean_gone`：清理远程已删除的本地分支；
  - 避免提交 `.env` 等敏感文件。

适合日常开发中减少重复 Git 操作。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/commit-commands

---

### 7. `claude-code-setup`

- **类型**：官方插件 / Skill
- **推荐指数**：★★★★☆
- **适合场景**：新项目初始化、存量项目接入 Claude Code
- **主要用途**：
  - 扫描代码库；
  - 推荐合适的 MCP；
  - 推荐 skills；
  - 推荐 hooks；
  - 推荐 subagents；
  - 推荐 slash commands；
  - 帮助项目形成 Claude Code 使用规范。

如果你刚把一个项目交给 Claude Code 辅助开发，可以先跑这个。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-code-setup

---

### 8. `claude-md-management`

- **类型**：官方插件 / CLAUDE.md 管理
- **推荐指数**：★★★★☆
- **适合场景**：项目上下文沉淀、团队规范维护
- **主要用途**：
  - 审计 CLAUDE.md 是否过期；
  - 检查项目说明是否与代码现状一致；
  - 将会话经验沉淀进 CLAUDE.md；
  - 维护长期项目记忆。

对于一个持续迭代的复杂项目来说，CLAUDE.md 很重要。  
它就是 Claude Code 的“项目脑子”，脑子越清楚，干活越靠谱。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/claude-md-management

---

### 9. `hookify`

- **类型**：官方插件 / Hooks
- **推荐指数**：★★★★☆
- **适合场景**：安全护栏、自动化规则、危险操作拦截
- **主要用途**：
  - 用自然语言生成 Hook；
  - 阻止危险 shell 命令；
  - 拦截敏感文件修改；
  - 在特定操作前提醒；
  - 可配置 warn 或 block。

示例：

```text
/hookify Warn me when I use rm -rf commands
```

适合团队统一配置 Claude Code 的安全边界。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/hookify

---

### 10. `security-guidance`

- **类型**：官方插件 / Hooks
- **推荐指数**：★★★★☆
- **适合场景**：安全敏感项目、生产配置、密钥、K8s 权限
- **主要用途**：
  - 对敏感操作给出安全提醒；
  - 降低误改配置、误删资源、泄露密钥的风险；
  - 与 hooks 搭配使用更稳。

适合涉及 kubeconfig、CI/CD token、数据库密码、生产配置的项目。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/plugins/security-guidance

---

## 三、MCP Server 推荐清单

MCP 是 Claude Code 连接外部系统的关键能力。

可以把 MCP 理解为：  
**Claude Code 的外部工具接口。**

它可以让 Claude Code 访问 GitHub、GitLab、K8s、数据库、浏览器、文档、CI/CD 等系统。

---

### 1. GitHub MCP Server

- **来源**：GitHub 官方开源
- **推荐指数**：★★★★★
- **适合场景**：GitHub 仓库、Issue、PR、Actions、代码审查
- **主要用途**：
  - 读取仓库代码；
  - 管理 issue；
  - 分析 PR；
  - 查看 Actions 失败原因；
  - 检查安全告警；
  - 结合 Claude Code 做需求和缺陷处理。

如果代码放在 GitHub，这是最推荐的 MCP 之一。

参考地址：

- https://github.com/github/github-mcp-server

---

### 2. GitLab MCP

- **来源**：官方插件市场 external plugin / 社区
- **推荐指数**：★★★★☆
- **适合场景**：GitLab 仓库、MR、Issue、CI
- **主要用途**：
  - 分析 GitLab MR；
  - 读取 Issue；
  - 查看 CI 流水线；
  - 辅助代码审查；
  - 与团队 GitLab 工作流集成。

如果你们公司代码仓主要是 GitLab，可以优先考虑它。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins

---

### 3. Context7 MCP

- **来源**：Upstash 开源 / 官方市场 external plugin
- **推荐指数**：★★★★★
- **适合场景**：查询最新技术文档
- **主要用途**：
  - 获取最新框架文档；
  - 避免模型使用过时 API；
  - 查询 Spring、K8s、client-go、Istio、Envoy、Kafka 等技术文档；
  - 辅助代码生成和缺陷修复。

这是一个很实用的工具。  
尤其当你问 Claude Code 某个新版本 API 怎么用时，它能有效减少“AI 一本正经胡说八道”的概率。

参考地址：

- https://github.com/anthropics/claude-plugins-official/blob/main/external_plugins/context7/.mcp.json
- https://github.com/upstash/context7

---

### 4. Kubernetes MCP Server

- **来源**：GitHub 开源
- **推荐指数**：★★★★★
- **适合场景**：K8s 集群诊断、资源查看、网络问题排查
- **主要用途**：
  - 查看 Deployment、Pod、Service、Endpoint、EndpointSlice；
  - 分析 K8s 资源状态；
  - 辅助排查容器网络问题；
  - 支持 read-only 模式；
  - 可用于 OpenShift / Kubernetes 诊断。

对于 Go / K8s 网络代理项目，这个非常值得接入。

建议默认开启只读模式，先让 Claude Code 分析，不要一上来就给它生产集群写权限。  
AI 很聪明，但生产环境更脆弱。

参考地址：

- https://github.com/containers/kubernetes-mcp-server

---

### 5. Serena

- **来源**：GitHub 开源
- **推荐指数**：★★★★★
- **适合场景**：大代码库语义检索、代码理解、重构
- **主要用途**：
  - 按符号理解代码；
  - 语义级代码检索；
  - 编辑和重构；
  - 辅助调试；
  - 通过 MCP 接入 Claude Code。

对于大型 Java 多模块项目和 Go 项目，Serena 非常适合。  
它能让 Claude Code 更像一个真正会用 IDE 的同事。

参考地址：

- https://github.com/oraios/serena

---

### 6. Playwright MCP

- **来源**：官方市场 external plugin
- **推荐指数**：★★★★☆
- **适合场景**：前端页面联调、E2E 测试、浏览器自动化
- **主要用途**：
  - 操作浏览器；
  - 验证页面行为；
  - 辅助 E2E 测试；
  - 检查页面交互；
  - 联调前后端功能。

如果你的项目包含日志检索页面、数据表格、可视化图表，Playwright MCP 很有价值。

参考地址：

- https://github.com/anthropics/claude-plugins-official/blob/main/external_plugins/playwright/.mcp.json

---

### 7. Terraform MCP

- **来源**：官方市场 external plugin / 社区
- **推荐指数**：★★★★☆
- **适合场景**：K8s、云资源、网络资源、IaC
- **主要用途**：
  - 分析 Terraform 配置；
  - 辅助修改云资源定义；
  - 检查基础设施变更风险；
  - 配合 K8s 部署和网络组件使用。

如果你的 K8s 网络代理涉及云资源、VPC、LB、安全组等内容，可以考虑使用。

参考地址：

- https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins

---

### 8. Filesystem MCP

- **来源**：MCP 官方 / 社区常用
- **推荐指数**：★★★★☆
- **适合场景**：文件访问、目录遍历、跨客户端工具集成
- **主要用途**：
  - 读取文件；
  - 遍历目录；
  - 搜索和替换；
  - 限制 Claude Code 可访问的工作目录。

Claude Code 本身已经具备较强的本地文件能力，但在跨客户端或自定义隔离目录时，Filesystem MCP 仍然有价值。

参考地址：

- https://github.com/modelcontextprotocol/servers

---

### 9. Git MCP / mcp-server-git

- **来源**：MCP 官方 / 社区
- **推荐指数**：★★★☆☆
- **适合场景**：本地 Git 历史、diff、branch 分析
- **主要用途**：
  - 查看 Git 历史；
  - 分析 diff；
  - 查看 branch；
  - 辅助定位缺陷引入时间点。

注意：Git MCP 曾出现过安全风险，建议使用新版本，并限制权限。

参考地址：

- https://pypi.org/project/mcp-server-git/
- https://github.com/modelcontextprotocol/servers

---

### 10. PostgreSQL / MySQL MCP

- **来源**：社区开源
- **推荐指数**：★★★☆☆
- **适合场景**：Java 微服务查库、数据问题排查
- **主要用途**：
  - 只读查询数据库；
  - 辅助定位数据异常；
  - 分析表结构；
  - 结合业务代码排查问题。

强烈建议：

```text
默认只读，不要给生产数据库写权限。
```

不要让 AI 当线上 DBA。  
数据库会哭，值班的人也会哭。

---

### 11. Slack / Linear / Jira MCP

- **来源**：官方市场或社区
- **推荐指数**：★★★☆☆
- **适合场景**：需求上下文、团队协作、Issue 管理
- **主要用途**：
  - 读取需求描述；
  - 分析 bug 工单；
  - 总结事故讨论；
  - 将 issue 信息转化为开发计划。

适合团队协作较成熟、需求和缺陷都沉淀在平台中的场景。

---

## 四、Subagents 推荐清单

如果不想一开始就装很多社区 agent，建议先使用官方插件里自带的 subagents。

---

### 1. `code-explorer`

- **来源**：`feature-dev`
- **推荐场景**：新需求开始前分析代码
- **适合用途**：
  - 梳理目录结构；
  - 找入口；
  - 查调用链；
  - 分析数据流；
  - 理解核心模块。

适合分析：

- Spring Controller → Service → DAO；
- Kafka Consumer → Service → Elasticsearch；
- Go Controller → Informer → Reconciler；
- K8s Proxy → iptables / eBPF / Service 规则生成逻辑。

---

### 2. `code-architect`

- **来源**：`feature-dev`
- **推荐场景**：复杂需求设计
- **适合用途**：
  - 给出多套实现方案；
  - 分析侵入性；
  - 评估风险；
  - 拆分任务；
  - 设计模块边界。

常见方案风格：

- 最小改动方案；
- 干净架构方案；
- 折中演进方案。

---

### 3. `code-reviewer`

- **来源**：`feature-dev` / `pr-review-toolkit`
- **推荐场景**：代码修改完成后审查
- **适合用途**：
  - 检查 bug；
  - 检查风格；
  - 检查架构一致性；
  - 检查异常处理；
  - 检查测试遗漏。

建议每次提交前都跑一遍。

---

### 4. `pr-test-analyzer`

- **来源**：`pr-review-toolkit`
- **推荐场景**：测试覆盖检查
- **适合用途**：
  - 检查测试是否覆盖核心逻辑；
  - 检查边界条件；
  - 检查失败路径；
  - 检查测试是否只是“凑数”。

适合 Java JUnit、Mockito、Testcontainers，也适合 Go unit test / integration test / e2e test。

---

### 5. `silent-failure-hunter`

- **来源**：`pr-review-toolkit`
- **推荐场景**：缺陷修复、安全排查、稳定性检查
- **适合用途**：
  - 检查 catch 后吞异常；
  - 检查 fallback 乱用；
  - 检查错误日志缺失；
  - 检查返回值未处理；
  - 检查失败路径没有告警。

Java 微服务和 Go 网络代理都很容易出现这类问题。  
这种 bug 最烦人：它不爆炸，它只是静静地把你坑了。

---

### 6. `code-simplifier`

- **来源**：`pr-review-toolkit`
- **推荐场景**：功能跑通后的代码简化
- **适合用途**：
  - 降低复杂度；
  - 减少重复逻辑；
  - 简化分支；
  - 提炼函数；
  - 改善可读性。

Claude Code 和人类一样，都有可能写出“能跑但像藤蔓一样缠绕”的代码。  
这个 agent 适合最后收尾时用。

---

### 7. 社区 Subagents 清单

如果想找更多社区 subagents，可以参考：

- https://github.com/VoltAgent/awesome-claude-code-subagents

该仓库收集了大量 Claude Code subagents，并提供交互式安装脚本。

建议不要一次性全装。  
更合理的方式是根据项目痛点挑 3～6 个：

- 代码审查类；
- 测试分析类；
- 安全检查类；
- K8s 排查类；
- 架构设计类；
- 文档沉淀类。

---

## 五、Skills 推荐清单

Skills 适合把你反复输入的工作流固化下来。

你可以把它理解为：

```text
把经验写成 Claude Code 能自动调用的说明书。
```

官方 Skill 一般通过 `SKILL.md` 组织，可以放在个人目录、项目目录或插件目录中。

---

### 推荐自定义 Skill 1：`spring-boot-debug`

适合 Java 微服务问题排查。

建议包含：

- Spring Boot 启动失败排查；
- Bean 注入问题；
- 配置加载问题；
- 事务问题；
- 连接池问题；
- 线程池问题；
- Kafka consumer lag；
- Elasticsearch 查询或写入问题；
- 日志链路追踪；
- 接口超时和重试排查。

---

### 推荐自定义 Skill 2：`java-microservice-pr-review`

适合 Java 微服务 PR 审查。

建议包含：

- 接口兼容性；
- 事务边界；
- 幂等性；
- 重试机制；
- 超时控制；
- 降级策略；
- 日志规范；
- 监控指标；
- 异常处理；
- 数据一致性；
- 并发安全；
- 性能风险。

---

### 推荐自定义 Skill 3：`go-k8s-controller-debug`

适合 Go / K8s controller、operator、网络代理项目。

建议包含：

- client-go 使用规范；
- informer / lister / watcher 排查；
- reconcile 逻辑检查；
- RBAC 权限排查；
- CRD schema 检查；
- leader election；
- context cancellation；
- goroutine 泄漏；
- rate limiter；
- workqueue 问题；
- Kubernetes API 调用失败分析。

---

### 推荐自定义 Skill 4：`k8s-network-debug`

适合 K8s 网络代理、CNI、Service 网络排查。

建议包含：

- Pod 网络连通性；
- Service 访问失败；
- DNS 解析问题；
- Endpoint / EndpointSlice；
- kube-proxy；
- iptables；
- ipvs；
- conntrack；
- eBPF；
- NodePort；
- LoadBalancer；
- NetworkPolicy；
- MTU；
- 跨节点通信；
- 日志和抓包分析。

---

### 推荐自定义 Skill 5：`log-service-debug`

适合你的日志服务类项目。

建议包含：

- log-agent 本地日志采集；
- 文件 offset 管理；
- Kafka 生产者发送失败；
- Kafka consumer lag；
- Elasticsearch bulk 写入；
- index template；
- mapping 冲突；
- 查询 DSL；
- 日志检索模板；
- 前端柱状图接口；
- 日志表格分页；
- 多服务配置管理；
- 部署脚本检查。

---

### 推荐自定义 Skill 6：`release-checklist`

适合发布前检查。

建议包含：

- 单元测试；
- 集成测试；
- 配置检查；
- 镜像构建；
- Helm / YAML 检查；
- 数据库变更；
- 向后兼容；
- 灰度发布；
- 回滚方案；
- 监控告警；
- 日志观察；
- 版本说明。

---

### 推荐自定义 Skill 7：`incident-postmortem`

适合线上故障复盘。

建议包含：

- 故障现象；
- 影响范围；
- 时间线；
- 初步定位；
- 根因分析；
- 修复措施；
- 验证结果；
- 遗留风险；
- 预防措施；
- 后续任务。

---

### 社区 Skills 清单

可以参考：

- https://github.com/ComposioHQ/awesome-claude-skills
- https://github.com/travisvn/awesome-claude-skills

不过在团队项目中，最有价值的通常不是通用 Skill，而是你们自己项目沉淀出来的 Skill。

通用 Skill 像菜谱，项目 Skill 才像你家厨房的灶台说明书。

---

## 六、Hooks 推荐清单

Hooks 适合给 Claude Code 加自动化动作和安全护栏。

---

### 1. 保存后自动格式化

Java 项目可以执行：

```bash
mvn spotless:apply
```

或：

```bash
gradle spotlessApply
```

Go 项目可以执行：

```bash
gofmt -w .
goimports -w .
```

作用：

- 统一格式；
- 减少无意义 diff；
- 保持提交干净。

---

### 2. 提交前自动测试

Java：

```bash
mvn test
```

Go：

```bash
go test ./...
```

作用：

- 防止明显错误进入提交；
- 让 Claude Code 修改代码后自动验证；
- 降低“看起来对，跑起来炸”的概率。

---

### 3. 阻止危险命令

建议拦截：

```bash
rm -rf /
kubectl delete ns
kubectl delete all --all
kubectl apply -f xxx --context prod
```

作用：

- 防止误删；
- 防止对生产环境做危险操作；
- 防止 Claude Code 被错误提示词带偏。

---

### 4. 阻止敏感文件编辑

建议保护：

```text
.env
kubeconfig
id_rsa
credentials
application-prod.yml
application-prod.yaml
secrets.yaml
```

作用：

- 防止泄露密钥；
- 防止误改生产配置；
- 防止把敏感文件提交到仓库。

---

### 5. 禁止硬编码密钥

检查关键词：

```text
AK
SK
token
password
secret
private_key
access_key
```

作用：

- 减少安全风险；
- 避免临时调试代码变成永久安全漏洞。

---

### 6. K8s 生产环境二次确认

当命令中出现以下内容时提醒：

```text
prod
production
kube-system
default
delete
apply
patch
scale
```

作用：

- 防止误操作；
- 增加生产环境保护；
- 让 Claude Code 先分析，再建议人工确认执行。

---

### 7. 会话结束前要求总结

建议让 Claude Code 在工作完成后输出：

- 修改了哪些文件；
- 为什么这样改；
- 运行了哪些测试；
- 是否还有未验证点；
- 存在哪些风险；
- 后续建议做什么。

这对团队协作和代码 review 很有帮助。

---

## 七、按开发场景推荐组合

### 1. Java 微服务后端项目推荐组合

```text
jdtls-lsp
github 或 gitlab
context7
feature-dev
code-review
pr-review-toolkit
commit-commands
claude-md-management
hookify
```

适用场景：

- Spring Boot 需求开发；
- 接口变更；
- Kafka 消费问题；
- Elasticsearch 写入和查询问题；
- 数据库缺陷修复；
- PR 审查；
- 提交前检查；
- 项目上下文沉淀。

---

### 2. Go + K8s 网络代理项目推荐组合

```text
gopls-lsp
kubernetes-mcp-server
github 或 gitlab
context7
serena
feature-dev
pr-review-toolkit
hookify
terraform
```

适用场景：

- Go client-go 项目；
- controller / operator；
- CNI / 网络代理；
- Service / EndpointSlice；
- iptables / ipvs / eBPF；
- K8s YAML / Helm / Terraform；
- 集群资源诊断；
- 网络问题排查。

---

### 3. 日志服务类项目推荐组合

```text
jdtls-lsp
gopls-lsp
github 或 gitlab
context7
feature-dev
code-review
pr-review-toolkit
claude-md-management
hookify
playwright
```

适用场景：

- log-agent 本地日志采集；
- Kafka 数据传输；
- Java consumer 写入 Elasticsearch；
- 后端查询服务；
- 前端日志柱状图和表格；
- 搜索模板；
- 配置管理服务；
- 前后端联调。

如果涉及数据库排查，可以补充 PostgreSQL / MySQL MCP。  
如果涉及 K8s 部署，可以补充 Kubernetes MCP 和 Terraform MCP。

---

## 八、安装与使用建议

### 1. 不要一次性安装太多

Claude Code 扩展不是越多越好。

建议先按以下顺序安装：

```text
语言 LSP
→ GitHub / GitLab MCP
→ Context7
→ feature-dev
→ code-review / pr-review-toolkit
→ claude-md-management
→ hookify
→ 其他项目专用 MCP
```

太多工具会让 Claude Code 的选择空间变大，反而增加混乱。

---

### 2. 优先只读，再逐步开放写权限

尤其是以下系统：

- GitHub / GitLab；
- Kubernetes；
- 数据库；
- Terraform；
- CI/CD；
- 生产配置。

推荐原则：

```text
默认只读 > 最小权限 > 项目级配置 > 生产环境禁写 > 敏感命令 hook 拦截
```

先让 Claude Code 分析，再让它修改。  
先让它给方案，再让人确认执行。  
不要一上来就把生产环境钥匙塞给它，不然它可能比实习生还“勇敢”。

---

### 3. 每个项目维护自己的 CLAUDE.md

建议在项目根目录维护：

```text
CLAUDE.md
```

内容可以包括：

- 项目简介；
- 模块划分；
- 技术栈；
- 启动方式；
- 测试方式；
- 编码规范；
- 提交流程；
- 常见问题；
- 禁止事项；
- 架构说明；
- 部署说明；
- 排查手册。

这能显著提高 Claude Code 对项目的理解能力。

---

### 4. 把重复流程做成 Skill

例如你经常这样问：

```text
帮我分析这个 Spring Boot 接口的调用链，找出可能的事务问题和异常处理缺陷。
```

那就可以做成一个 skill。

例如你经常这样问：

```text
帮我排查这个 K8s Service 无法访问的问题，从 DNS、Endpoint、iptables、Pod 网络逐步分析。
```

那也可以做成一个 skill。

重复三次以上的提示词，基本就值得沉淀。

---

### 5. 把危险操作做成 Hook

以下规则建议尽早加上：

- 阻止修改 `.env`；
- 阻止提交密钥；
- 阻止 `rm -rf`；
- 阻止生产 kubeconfig 写操作；
- 阻止删除 namespace；
- 提交前自动测试；
- 修改后自动格式化。

这些规则不是为了限制 Claude Code，而是为了保护你自己。  
毕竟锅不会因为是 AI 改的就自动变轻。

---

## 九、推荐优先级总结

### 必装级

```text
jdtls-lsp
gopls-lsp
feature-dev
code-review
pr-review-toolkit
context7
GitHub MCP / GitLab MCP
claude-md-management
```

### 强烈推荐

```text
hookify
security-guidance
kubernetes-mcp-server
serena
commit-commands
```

### 按需安装

```text
playwright
terraform
postgresql / mysql mcp
slack / jira / linear mcp
filesystem mcp
git mcp
```

---

## 十、最终建议

如果你是 Java 微服务后端开发人员，同时也在做 Go / K8s 网络代理工具开发，我建议先从下面这套组合开始：

```text
jdtls-lsp
gopls-lsp
GitHub MCP 或 GitLab MCP
Context7 MCP
feature-dev
code-review
pr-review-toolkit
claude-md-management
hookify
kubernetes-mcp-server
serena
```

这套组合覆盖了：

- Java 代码理解；
- Go 代码理解；
- 大代码库语义分析；
- Git 仓库协作；
- 最新文档查询；
- K8s 集群诊断；
- 需求开发；
- 缺陷修复；
- PR 审查；
- 项目记忆沉淀；
- 安全护栏。

对于大部分后端开发者来说，这已经非常够用。

真正重要的不是装了多少插件，而是形成稳定流程：

```text
先理解项目
→ 再分析需求
→ 再设计方案
→ 再小步修改
→ 再运行测试
→ 再审查代码
→ 最后沉淀经验
```

Claude Code 用得好的关键，不是把它当“代码生成器”，而是把它当作一个可以陪你一起分析、开发、复盘、沉淀的工程搭档。
