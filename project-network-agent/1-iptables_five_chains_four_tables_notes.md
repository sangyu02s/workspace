# iptables、五链、四表核心笔记

> 适合 Linux 网络、Kubernetes 网络、服务转发、网络代理项目入门使用。  
> 本笔记只保留重点，重点关注：`nat`、`mangle`、地址转换、打标记、包路径。

---

## 1. iptables 是什么

**iptables 是 Linux 上配置网络包处理规则的工具。**

它用于告诉 Linux：

```text
匹配什么样的网络包？
匹配后执行什么动作？
```

<span style="color:red">重点：</span>

```text
iptables：负责写规则
netfilter：负责在 Linux 内核中执行规则
```

iptables 本身不是代理程序，也不是长期运行的服务。  
它更像是一个“规则配置工具”。

---

## 2. iptables 处理对象

iptables 处理的是 **网络包 packet**。

常见匹配字段：

| 匹配项 | 含义 |
|---|---|
| 源 IP | 包从哪里来 |
| 目标 IP | 包要去哪里 |
| 源端口 | 来源端口 |
| 目标端口 | 访问端口 |
| 协议 | TCP / UDP / ICMP |
| 入接口 | 从哪个网卡进入 |
| 出接口 | 从哪个网卡发出 |
| mark | 包的内部标记 |

<span style="color:red">重点：</span>iptables 主要处理三层 / 四层网络信息，不理解业务语义。

---

## 3. iptables 常见动作

| 动作 | 含义 |
|---|---|
| `ACCEPT` | 放行 |
| `DROP` | 丢弃，不通知对方 |
| `REJECT` | 拒绝，并通知对方 |
| `DNAT` | 修改目标 IP / 端口 |
| `SNAT` | 修改源 IP / 端口 |
| `MASQUERADE` | 动态 SNAT |
| `MARK` | 给包打内部标记 |

---

## 4. 五链概览

五链表示：**网络包在 Linux 中经过的 5 个处理位置**。

| 链 | 位置 | 作用 |
|---|---|---|
| `PREROUTING` | 路由判断前 | 外部包刚进入机器 |
| `INPUT` | 进入本机 | 目标是本机进程 |
| `FORWARD` | 转发过程 | 经过本机转发 |
| `OUTPUT` | 本机发出 | 本机进程产生的包 |
| `POSTROUTING` | 路由判断后 | 包离开机器前 |

<span style="color:red">重点口诀：</span>

```text
PREROUTING：刚进来，路由前
INPUT：给本机
FORWARD：路过本机
OUTPUT：本机发出
POSTROUTING：快出去了
```

---

## 5. 三条典型包路径

### 5.1 外部访问本机

```text
外部客户端
  ↓
PREROUTING
  ↓
路由判断
  ↓
INPUT
  ↓
本机进程
```

---

### 5.2 外部包经过本机转发

```text
外部客户端
  ↓
PREROUTING
  ↓
路由判断
  ↓
FORWARD
  ↓
POSTROUTING
  ↓
发出
```

---

### 5.3 本机进程主动发包

```text
本机进程
  ↓
OUTPUT
  ↓
路由判断
  ↓
POSTROUTING
  ↓
发出
```

<span style="color:red">重点：</span>本机进程发出的包不走 `PREROUTING`，而是走 `OUTPUT`。

---

## 6. 四表概览

四表表示：**iptables 按用途划分的 4 类规则表**。

| 表 | 作用 | 常见动作 |
|---|---|---|
| `filter` | 过滤包 | `ACCEPT` / `DROP` / `REJECT` |
| `nat` | 地址转换 | `DNAT` / `SNAT` / `MASQUERADE` / `REDIRECT` |
| `mangle` | 修改包属性 / 打标记 | `MARK` / `CONNMARK` |
| `raw` | conntrack 前处理 | `NOTRACK` |

<span style="color:red">重点：</span>

```text
filter：要不要让包过？
nat：要不要改 IP / 端口？
mangle：要不要打标记？
raw：要不要跳过 conntrack？
```

学习优先级：

```text
nat > mangle > filter > raw
```

---

## 7. nat 表

### 7.1 nat 表作用

`nat` 表负责 **地址转换**。

| 动作 | 含义 | 常见位置 |
|---|---|---|
| `DNAT` | 修改目标地址 | `PREROUTING` / `OUTPUT` |
| `SNAT` | 修改源地址 | `POSTROUTING` |
| `MASQUERADE` | 动态 SNAT | `POSTROUTING` |
| `REDIRECT` | 重定向到本机端口 | `PREROUTING` / `OUTPUT` |

---

### 7.2 DNAT

DNAT 修改的是：

```text
目标 IP
目标端口
```

常用于：

```text
端口转发
服务转发
虚拟 IP 转真实 IP
```

<span style="color:red">重点：</span>外部包做 DNAT 通常放在 `nat PREROUTING`，因为要在路由判断前修改目标地址。

本机进程发出的包不走 `PREROUTING`，如果需要对本机发包做 DNAT，通常看：

```text
nat OUTPUT
```

---

### 7.3 SNAT / MASQUERADE

SNAT 修改的是：

```text
源 IP
源端口
```

常用于：

```text
出口访问
隐藏内部地址
保证回包路径正确
```

`MASQUERADE` 可以理解为动态 SNAT：

```text
根据出口网卡自动选择源 IP
```

<span style="color:red">重点：</span>SNAT / MASQUERADE 通常放在 `nat POSTROUTING`，因为要在包离开机器前修改源地址。

---

### 7.4 nat 表常见链

```text
nat PREROUTING：外部包 DNAT
nat OUTPUT：本机发包 DNAT
nat POSTROUTING：SNAT / MASQUERADE
```

---

## 8. mangle 表

### 8.1 mangle 表作用

`mangle` 表用于：

```text
修改包属性
给包打 mark
配合策略路由
```

入门阶段最重要的是：

```text
MARK
```

---

### 8.2 MARK

MARK 会给包设置 Linux 内核内部标记。

示例：

```text
mark = 0x100
```

这个标记不会发到对端机器，只在本机内核中使用。

典型流程：

```text
iptables mangle 打 MARK
  ↓
ip rule 根据 fwmark 匹配
  ↓
选择指定路由表
  ↓
ip route 决定出口
```

<span style="color:red">重点：</span>mangle 表常用于 `MARK + ip rule + ip route` 组合。

---

### 8.3 mangle 表常见链

```text
mangle PREROUTING：外部包路由前打 MARK
mangle OUTPUT：本机发包路由前打 MARK
mangle FORWARD：转发过程中处理
mangle POSTROUTING：发出前处理
```

---

## 9. 五链、包时机、nat/mangle 总览表

<span style="color:red">建议重点背这一张表。</span>

| 包路径阶段 | 五链 | 包处理时机 | `nat` 表常见动作 | `mangle` 表常见动作 | 常见用途 | 排查关注点 |
|---|---|---|---|---|---|---|
| 外部包刚进入机器 | `PREROUTING` | 路由判断前 | `DNAT` | `MARK` | 外部流量改目标地址；路由前打标记 | 包是否进入本机；DNAT/MARK 计数是否增长 |
| 目标是本机 | `INPUT` | 路由判断后，交给本机进程前 | 较少关注 | 较少关注 | 本机服务入站处理 | 是否是访问本机进程 |
| 经过本机转发 | `FORWARD` | 路由判断后，转发过程中 | 较少关注 | 可做特殊处理 | 转发流量检查 | 是否被 filter FORWARD 拦截 |
| 本机进程发包 | `OUTPUT` | 本机进程产生包后，路由判断前后 | `DNAT` | `MARK` | 本机访问目标时改写；本机发包打标记 | 本机访问目标时是否命中 OUTPUT |
| 包离开机器前 | `POSTROUTING` | 路由判断后，出网卡前 | `SNAT` / `MASQUERADE` | 较少关注 | 修改源地址；保证回包路径 | 是否需要 SNAT；回包路径是否正确 |

---

## 10. 极简记忆表

| 五链 | 时机 | 重点表 | 重点动作 |
|---|---|---|---|
| `PREROUTING` | 外部包刚进来，路由前 | `nat` / `mangle` | `DNAT` / `MARK` |
| `INPUT` | 目标是本机 | 暂不重点关注 | 入站过滤为主 |
| `FORWARD` | 经过本机转发 | 暂不重点关注 | 转发过滤为主 |
| `OUTPUT` | 本机进程发包 | `nat` / `mangle` | `DNAT` / `MARK` |
| `POSTROUTING` | 包离开机器前 | `nat` | `SNAT` / `MASQUERADE` |

---

## 11. 两张包路径图

### 11.1 外部包进入并转发

```text
网卡入口
  ↓
PREROUTING
  ├─ mangle：MARK
  └─ nat：DNAT
  ↓
路由判断
  ↓
FORWARD
  ↓
POSTROUTING
  └─ nat：SNAT / MASQUERADE
  ↓
网卡出口
```

---

### 11.2 本机进程发包

```text
本机进程
  ↓
OUTPUT
  ├─ mangle：MARK
  └─ nat：DNAT
  ↓
路由判断
  ↓
POSTROUTING
  └─ nat：SNAT / MASQUERADE
  ↓
网卡出口
```

---

## 12. 四表五链常用组合

| 常用组合 | 作用 |
|---|---|
| `nat PREROUTING` | 外部包 DNAT |
| `nat OUTPUT` | 本机发包 DNAT |
| `nat POSTROUTING` | SNAT / MASQUERADE |
| `mangle PREROUTING` | 外部包打 MARK |
| `mangle OUTPUT` | 本机发包打 MARK |
| `filter INPUT` | 控制访问本机 |
| `filter FORWARD` | 控制转发流量 |
| `filter OUTPUT` | 控制本机出站 |

---

## 13. 常见查看命令

查看所有规则：

```bash
iptables-save
```

查看 nat 表：

```bash
iptables-save -t nat
iptables -t nat -L -n -v --line-numbers
```

查看 mangle 表：

```bash
iptables-save -t mangle
iptables -t mangle -L -n -v --line-numbers
```

查看 filter 表：

```bash
iptables-save -t filter
iptables -t filter -L -n -v --line-numbers
```

参数说明：

| 参数 | 含义 |
|---|---|
| `-n` | 不解析域名，直接显示 IP / 端口 |
| `-v` | 显示详细信息，包括包计数 |
| `--line-numbers` | 显示规则行号 |

---

## 14. 排查时重点看什么

| 关注点 | 说明 |
|---|---|
| 表 | 规则属于 `nat`、`mangle` 还是 `filter` |
| 链 | 包是否真的会经过这条链 |
| 匹配条件 | IP、端口、协议是否正确 |
| 动作 | DNAT、SNAT、MARK、DROP 等 |
| 计数 | `pkts` / `bytes` 是否增长 |

<span style="color:red">重点：</span>规则计数不增长，通常说明包没有经过该规则，或者匹配条件不正确。

---

## 15. 最重要结论

```text
iptables：配置 Linux 网络包处理规则
五链：包经过 Linux 的处理位置
四表：规则按用途分类
nat：负责地址转换，重点是 DNAT / SNAT / MASQUERADE
mangle：负责打标记，重点是 MARK
```

<span style="color:red">最重要：</span>

```text
DNAT：重点看 nat PREROUTING / nat OUTPUT
SNAT：重点看 nat POSTROUTING
MARK：重点看 mangle PREROUTING / mangle OUTPUT
转发是否允许：常看 filter FORWARD
```

<span style="color:red">一句话记忆：</span>

```text
外部来的包，看 PREROUTING；
本机发出的包，看 OUTPUT；
快出机器的包，看 POSTROUTING。
```
