# Linux NAT、conntrack、MARK 与策略路由核心笔记

> 适合 Linux 网络、容器网络、服务转发、网络代理项目入门复习。  
> 本笔记只保留重点，重点关注：`DNAT`、`SNAT/MASQUERADE`、`conntrack`、`MARK`、`ip rule`、`ip route`。

---

## 1. 总览

| 术语 | 作用 | 是否改包 IP/端口 | 常见位置 / 命令 |
|---|---|---|---|
| `DNAT` | 修改目标地址 | 改目标 IP / 端口 | `nat PREROUTING` / `nat OUTPUT` |
| `SNAT` | 修改源地址 | 改源 IP / 端口 | `nat POSTROUTING` |
| `MASQUERADE` | 动态 SNAT | 改源 IP / 端口 | `nat POSTROUTING` |
| `conntrack` | 记录连接与 NAT 映射 | 不直接改规则 | `conntrack -L` / `conntrack -E` |
| `MARK` | 给包打内部标记 | 不改 IP / 端口 | `mangle PREROUTING` / `mangle OUTPUT` |
| `ip rule` | 根据条件选择路由表 | 不改 IP / 端口 | `ip rule show` |
| `ip route` | 决定出口和下一跳 | 不改 IP / 端口 | `ip route show` |

<span style="color:red">重点：</span>

```text
DNAT / SNAT：负责改地址
conntrack：负责记住改写关系
MARK：负责给包分类
ip rule：负责按分类选择路由表
ip route：负责决定包怎么走
```

---

## 2. DNAT

### 2.1 是什么

`DNAT = Destination NAT`

中文：**目标地址转换**

它修改的是：

```text
目标 IP
目标端口
```

示意：

```text
修改前：
client-ip:client-port -> old-ip:old-port

修改后：
client-ip:client-port -> new-ip:new-port
```

<span style="color:red">重点：</span>DNAT 只改目标地址，不改源地址。

---

### 2.2 常见位置

| 场景 | 常见链 |
|---|---|
| 外部包进入本机 | `nat PREROUTING` |
| 本机进程发包 | `nat OUTPUT` |

原因：

```text
PREROUTING：外部包路由判断前
OUTPUT：本机进程发包时
```

<span style="color:red">重点：</span>外部包做 DNAT，通常放在 `nat PREROUTING`，因为要在路由判断前改目标地址。

---

### 2.3 典型命令结构

```bash
iptables -t nat -A PREROUTING \
  -d <old-ip> -p <tcp|udp> --dport <old-port> \
  -j DNAT --to-destination <new-ip>:<new-port>
```

---

### 2.4 包路径图

```text
外部包进入
  ↓
nat PREROUTING
  └─ DNAT：old-ip:old-port -> new-ip:new-port
  ↓
路由判断
  ↓
FORWARD / INPUT
  ↓
POSTROUTING
  ↓
发出 / 交给本机
```

---

## 3. SNAT / MASQUERADE

### 3.1 SNAT 是什么

`SNAT = Source NAT`

中文：**源地址转换**

它修改的是：

```text
源 IP
源端口
```

示意：

```text
修改前：
old-src-ip:old-src-port -> server-ip:server-port

修改后：
new-src-ip:new-src-port -> server-ip:server-port
```

<span style="color:red">重点：</span>SNAT 只改源地址，不改目标地址。

---

### 3.2 MASQUERADE 是什么

`MASQUERADE` 可以理解为：**动态 SNAT**

区别：

| 对比项 | SNAT | MASQUERADE |
|---|---|---|
| 源 IP | 手动指定 | 自动使用出口网卡 IP |
| 适合场景 | 出口 IP 固定 | 出口 IP 可能变化 |
| 常见链 | `nat POSTROUTING` | `nat POSTROUTING` |

<span style="color:red">重点：</span>出口 IP 固定优先用 `SNAT`，出口 IP 动态常用 `MASQUERADE`。

---

### 3.3 常见位置

```text
SNAT / MASQUERADE：nat POSTROUTING
```

原因：

```text
POSTROUTING 是路由判断后、包离开机器前；
此时 Linux 已经知道包要从哪个网卡出去。
```

---

### 3.4 典型命令结构

SNAT：

```bash
iptables -t nat -A POSTROUTING \
  -s <old-src-cidr> \
  -j SNAT --to-source <new-src-ip>
```

MASQUERADE：

```bash
iptables -t nat -A POSTROUTING \
  -s <old-src-cidr> -o <out-dev> \
  -j MASQUERADE
```

---

### 3.5 包路径图

```text
包准备发出
  ↓
路由判断完成
  ↓
nat POSTROUTING
  └─ SNAT / MASQUERADE：old-src -> new-src
  ↓
网卡出口
```

<span style="color:red">重点：</span>SNAT 经常用于保证回包路径正确。

---

## 4. DNAT / SNAT / MASQUERADE 对比

| 项目 | DNAT | SNAT | MASQUERADE |
|---|---|---|---|
| 修改对象 | 目标 IP / 端口 | 源 IP / 端口 | 源 IP / 端口 |
| 常见表 | `nat` | `nat` | `nat` |
| 常见链 | `PREROUTING` / `OUTPUT` | `POSTROUTING` | `POSTROUTING` |
| 典型用途 | 转发到新目标 | 保证回包路径 / 隐藏源地址 | 动态出口 NAT |
| 是否指定 IP | 指定新目标 IP | 指定新源 IP | 自动选择出口 IP |

<span style="color:red">一句话：</span>

```text
DNAT：改目标，让包去正确的地方
SNAT：改源，让回包回到正确的地方
MASQUERADE：动态 SNAT，自动使用出口网卡 IP
```

---

## 5. conntrack

### 5.1 是什么

`conntrack = connection tracking`

中文：**连接跟踪**

它是 Linux 内核维护的一张连接状态表，用来记录：

```text
协议
源 IP / 源端口
目标 IP / 目标端口
连接状态
NAT 映射关系
超时时间
```

<span style="color:red">重点：</span>DNAT / SNAT / MASQUERADE 能正确处理回包，依赖 `conntrack`。

---

### 5.2 conntrack 与 DNAT

DNAT 请求方向：

```text
client -> old-ip
  ↓ DNAT
client -> backend-ip
```

回包方向：

```text
backend-ip -> client
  ↓ conntrack 还原
old-ip -> client
```

<span style="color:red">重点：</span>DNAT 改请求包的目标地址，conntrack 负责让回包源地址被还原。

---

### 5.3 conntrack 与 SNAT

SNAT 请求方向：

```text
internal-ip -> server
  ↓ SNAT
node-ip -> server
```

回包方向：

```text
server -> node-ip
  ↓ conntrack 还原
server -> internal-ip
```

<span style="color:red">重点：</span>SNAT 改请求包的源地址，conntrack 负责让回包目标地址被还原。

---

### 5.4 TCP / UDP 都会被跟踪

| 协议 | conntrack 行为 |
|---|---|
| TCP | 根据连接状态跟踪，例如 `NEW` / `ESTABLISHED` |
| UDP | 没有真实连接，但会创建“伪连接记录”，依赖超时时间清理 |

---

### 5.5 常见状态

| 状态 | 含义 |
|---|---|
| `NEW` | 新连接 |
| `ESTABLISHED` | 已建立连接 |
| `RELATED` | 与已有连接相关 |
| `INVALID` | 异常包 |

---

### 5.6 常用命令

```bash
conntrack -L              # 查看连接跟踪表
conntrack -L -p tcp       # 查看 TCP
conntrack -L -p udp       # 查看 UDP
conntrack -E              # 实时监听连接事件
conntrack -L --dst-nat    # 查看 DNAT 连接
conntrack -L --src-nat    # 查看 SNAT 连接
```

查看表容量：

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

---

### 5.7 常见问题

| 问题 | 可能原因 |
|---|---|
| 规则改了但流量还走旧路径 | 旧 conntrack 记录仍在 |
| 新连接失败 | conntrack 表满 |
| UDP 回包异常 | UDP conntrack 超时 |
| 请求到达后端但客户端无响应 | 回包未经过原 NAT 节点 |

<span style="color:red">重点：</span>做 NAT 的机器，通常必须看到回包，否则无法根据 conntrack 还原。

---

## 6. MARK

### 6.1 是什么

`MARK` 是 iptables 的一个动作，用于给包打 Linux 内核内部标记。

示意：

```text
mark = 0x100
```

<span style="color:red">重点：</span>MARK 不会发到对端机器，只在当前 Linux 内核中有效。

---

### 6.2 常见位置

| 场景 | 常见链 |
|---|---|
| 外部包进入 | `mangle PREROUTING` |
| 本机进程发包 | `mangle OUTPUT` |

原因：

```text
要在路由判断前打 MARK，后续 ip rule 才能根据 mark 选择路由表。
```

---

### 6.3 典型命令结构

外部包：

```bash
iptables -t mangle -A PREROUTING \
  <匹配条件> \
  -j MARK --set-mark 0x100
```

本机包：

```bash
iptables -t mangle -A OUTPUT \
  <匹配条件> \
  -j MARK --set-mark 0x100
```

---

## 7. ip rule

### 7.1 是什么

`ip rule` 是 Linux 的**策略路由规则**。

它负责：

```text
根据条件选择哪张路由表
```

常见匹配条件：

```text
源 IP
目标 IP
入接口
fwmark
协议
端口
```

入门阶段重点关注：

```text
fwmark
```

---

### 7.2 典型命令结构

```bash
ip rule add fwmark 0x100 table 100 priority 100
```

含义：

```text
如果包带有 fwmark 0x100，
就查 table 100 这张路由表。
```

字段说明：

| 字段 | 含义 |
|---|---|
| `fwmark 0x100` | 匹配包上的 mark |
| `table 100` | 使用 100 号路由表 |
| `priority 100` | 规则优先级，数字越小优先级越高 |

<span style="color:red">重点：</span>MARK 本身只是分类，真正改变路由选择的是 `ip rule`。

---

## 8. ip route

### 8.1 是什么

`ip route` 是 Linux 路由表配置。

它负责回答：

```text
目标 IP 从哪个网卡出去？
下一跳是谁？
使用哪个源 IP？
```

---

### 8.2 常用命令

查看默认主路由表：

```bash
ip route
```

查看所有路由表：

```bash
ip route show table all
```

查看指定路由表：

```bash
ip route show table 100
```

添加特殊路由：

```bash
ip route add default via <gateway> dev <dev> table 100
```

测试普通路由：

```bash
ip route get <target-ip>
```

测试带 mark 的路由：

```bash
ip route get <target-ip> mark 0x100
```

---

## 9. MARK + ip rule + ip route

### 9.1 核心流程

```text
iptables mangle MARK
  ↓
ip rule fwmark
  ↓
ip route table
```

解释：

```text
MARK：给包贴标签
ip rule：根据标签选择路由表
ip route：在该路由表里决定出口和下一跳
```

<span style="color:red">一句话：</span>

```text
MARK 负责分类；
ip rule 负责分流；
ip route 负责指路。
```

---

### 9.2 包路径图

外部包：

```text
外部包进入
  ↓
mangle PREROUTING
  └─ MARK：打 0x100
  ↓
ip rule
  └─ fwmark 0x100 -> table 100
  ↓
ip route table 100
  └─ 决定出口网卡 / 下一跳
  ↓
继续转发或交给本机
```

本机包：

```text
本机进程发包
  ↓
mangle OUTPUT
  └─ MARK：打 0x100
  ↓
ip rule
  └─ fwmark 0x100 -> table 100
  ↓
ip route table 100
  └─ 决定出口网卡 / 下一跳
  ↓
POSTROUTING
  ↓
发出
```

---

## 10. NAT 与策略路由对比

| 能力 | 作用 | 是否改 IP/端口 |
|---|---|---|
| `DNAT` | 改目标地址 | 是 |
| `SNAT` | 改源地址 | 是 |
| `MASQUERADE` | 动态改源地址 | 是 |
| `conntrack` | 记录连接和 NAT 映射 | 否 |
| `MARK` | 给包打标签 | 否 |
| `ip rule` | 根据条件选路由表 | 否 |
| `ip route` | 决定出口和下一跳 | 否 |

<span style="color:red">重点：</span>

```text
DNAT/SNAT/MASQUERADE：改变地址
MARK/ip rule/ip route：改变走哪条路
conntrack：记住连接与 NAT 映射
```

---

## 11. 常见排查命令

### 11.1 NAT 规则

```bash
iptables-save -t nat
iptables -t nat -L -n -v --line-numbers
```

重点看：

```text
DNAT 是否命中
SNAT/MASQUERADE 是否命中
pkts / bytes 是否增长
```

---

### 11.2 mangle / MARK 规则

```bash
iptables-save -t mangle
iptables -t mangle -L -n -v --line-numbers
```

重点看：

```text
MARK 规则是否存在
mark 值是否正确
规则计数是否增长
```

---

### 11.3 ip rule / ip route

```bash
ip rule show
ip route show table all
ip route show table 100
ip route get <target-ip>
ip route get <target-ip> mark 0x100
```

重点看：

```text
fwmark 是否对应
priority 是否合理
table 中是否有路由
出口 dev / 下一跳 via 是否正确
```

---

### 11.4 conntrack

```bash
conntrack -L
conntrack -E
conntrack -L -p tcp
conntrack -L -p udp
conntrack -L --dst-nat
conntrack -L --src-nat
```

重点看：

```text
是否创建连接记录
original / reply 方向是否符合预期
是否有旧记录影响新规则
```

---

### 11.5 抓包

```bash
tcpdump -ni any host <ip>
tcpdump -ni any port <port>
tcpdump -ni <dev> host <ip>
```

重点看：

```text
包是否进入本机
DNAT/SNAT 前后地址是否变化
包是否从预期网卡出去
回包是否经过本机
```

---

## 12. 排查顺序

### 12.1 NAT 问题

```text
1. 包有没有进本机？
2. nat 规则计数是否增长？
3. DNAT/SNAT 后地址是否符合预期？
4. 路由是否正确？
5. 回包是否经过原 NAT 节点？
6. conntrack 是否有记录？
7. 是否存在旧 conntrack 记录？
```

---

### 12.2 MARK + 策略路由问题

```text
1. 包有没有经过 mangle 表？
2. MARK 规则计数是否增长？
3. mark 值是否正确？
4. ip rule 是否匹配同一个 fwmark？
5. ip rule priority 是否合理？
6. 对应 table 是否存在路由？
7. ip route get <目标IP> mark <mark值> 是否符合预期？
8. tcpdump 看包实际从哪个接口出去
```

---

## 13. 最重要总结

```text
DNAT：改目标，让包去正确的地方
SNAT：改源，让回包回到正确的地方
MASQUERADE：动态 SNAT，自动使用出口网卡 IP
conntrack：记录连接与 NAT 映射，保证回包能还原
MARK：给包贴标签
ip rule：根据标签选择路由表
ip route：在路由表里决定出口和下一跳
```

<span style="color:red">最终记忆：</span>

```text
DNAT / SNAT 是“改地址”
conntrack 是“记关系”
MARK 是“贴标签”
ip rule 是“按标签选表”
ip route 是“查表指路”
```
