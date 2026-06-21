# IPVS 核心笔记

> 适合 Linux 四层负载均衡、Kubernetes Service 转发、网络代理项目入门使用。  
> 本笔记只保留重点，重点关注：`Virtual Server`、`Real Server`、`ipvsadm -Ln`、`rr`、`sh`、`Masq`、排查思路。

---

## 1. IPVS 是什么

**IPVS = IP Virtual Server**

可以理解为：

> **Linux 内核里的四层负载均衡器。**

它负责把访问某个虚拟服务的流量，转发到后端真实服务。

```text
client
  ↓
VIP:Port
  ↓
IPVS
  ↓
RealServerIP:Port
```

<span style="color:red">重点：</span>

```text
IPVS 负责 VIP:Port 到 RealServerIP:Port 的四层负载均衡与转发。
```

---

## 2. IPVS 核心模型

| 概念 | 含义 | 常见形式 |
|---|---|---|
| `Virtual Server` | 虚拟服务，对外入口 | `VIP:Port` |
| `Real Server` | 真实后端 | `PodIP:Port` / `ServerIP:Port` |
| `Scheduler` | 调度算法 | `rr` / `sh` / `wrr` / `lc` |
| `Forward Method` | 转发模式 | `Masq` / `Route` / `Tunnel` |

<span style="color:red">重点：</span>

```text
Virtual Server = 对外入口
Real Server = 真实后端
Scheduler = 怎么选择后端
Forward Method = 怎么转发到后端
```

---

## 3. IPVS 与四层负载均衡

IPVS 属于 **L4 Load Balancing**，主要关注：

```text
源 IP
源端口
目标 IP
目标端口
协议 TCP / UDP / SCTP
```

它一般不关心：

```text
HTTP URL
HTTP Header
JSON 字段
业务参数
```

<span style="color:red">重点：</span>

```text
IPVS 看 IP、端口、协议；不理解应用层业务内容。
```

---

## 4. IPVS 基本路径图

```text
client
  ↓
VIP:Port
  ↓
IPVS Virtual Server
  ↓ Scheduler 选择后端
Real Server 1: PodIP1:Port
Real Server 2: PodIP2:Port
Real Server 3: PodIP3:Port
```

---

## 5. Masq 模式路径图

`Masq` 通常可以理解为 **NAT / Masquerading 转发模式**。

```text
请求路径：

client
  ↓
VIP:Port
  ↓
IPVS 节点
  ↓
Real Server / PodIP:Port


回包路径：

Real Server / PodIP:Port
  ↓
IPVS 节点
  ↓
client
```

<span style="color:red">重点：</span>

```text
Masq 模式下，请求和回包通常都经过 IPVS 节点。
```

优点：

```text
网络要求相对低
后端不需要持有 VIP
回包路径容易控制
适合容器 / Pod 后端场景
```

缺点：

```text
请求和回包都经过 IPVS 节点
IPVS 节点可能成为流量瓶颈
```

---

## 6. ipvsadm -Ln 是什么

常用命令：

```bash
ipvsadm -Ln
```

含义：

| 参数 | 含义 |
|---|---|
| `-L` | list，列出 IPVS 规则 |
| `-n` | numeric，直接显示数字 IP 和端口，不做名称解析 |

<span style="color:red">重点：</span>

```text
ipvsadm -Ln 用于查看当前节点上的 IPVS 虚拟服务和真实后端。
```

---

## 7. ipvsadm -Ln 输出示例

```text
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.0.0.100:18999 rr
  -> 10.244.1.23:18999            Masq    1      0          5
  -> 10.244.2.45:18999            Masq    1      0          3
```

---

## 8. ipvsadm -Ln 字段说明

| 字段 | 含义 |
|---|---|
| `Prot` | 协议，如 `TCP`、`UDP` |
| `LocalAddress:Port` | 虚拟服务地址，即 `VIP:Port` |
| `Scheduler` | 调度算法，如 `rr`、`sh` |
| `Flags` | 虚拟服务标志，如持久连接等 |
| `RemoteAddress:Port` | 真实后端地址，即 `RealServerIP:Port` |
| `Forward` | 转发模式，如 `Masq` |
| `Weight` | 后端权重 |
| `ActiveConn` | 活跃连接数 |
| `InActConn` | 非活跃连接数 |

---

## 9. Virtual Server 行怎么看

示例：

```text
TCP  10.0.0.100:18999 rr
```

含义：

```text
协议：TCP
虚拟服务：10.0.0.100:18999
调度算法：rr
```

也就是：

```text
客户端访问 10.0.0.100:18999
会进入该 IPVS Virtual Server
IPVS 使用 rr 算法选择后端
```

<span style="color:red">重点：</span>

```text
看到 TCP/UDP VIP:Port，就是 Virtual Server。
```

---

## 10. Real Server 行怎么看

示例：

```text
  -> 10.244.1.23:18999            Masq    1      0          5
```

含义：

```text
真实后端：10.244.1.23:18999
转发模式：Masq
权重：1
活跃连接数：0
非活跃连接数：5
```

<span style="color:red">重点：</span>

```text
看到 -> PodIP:Port，就是 Real Server。
```

---

## 11. rr 调度算法

`rr = Round Robin`

中文：

```text
轮询
```

含义：

```text
第 1 个连接 -> 后端 A
第 2 个连接 -> 后端 B
第 3 个连接 -> 后端 C
第 4 个连接 -> 后端 A
```

适合场景：

```text
后端性能接近
请求处理成本接近
希望简单均匀分配
```

<span style="color:red">重点：</span>

```text
rr 追求简单、平均、按顺序分配。
```

---

## 12. sh 调度算法

`sh = Source Hashing`

中文：

```text
源地址哈希
```

含义：

```text
根据客户端源 IP 做 hash
hash 结果决定选择哪个 Real Server
```

示意：

```text
clientA -> hash(clientA) -> 后端 1
clientB -> hash(clientB) -> 后端 2
clientC -> hash(clientC) -> 后端 1
```

适合场景：

```text
希望同一个客户端尽量固定到同一个后端
需要一定客户端亲和性
UDP 流量希望同源更稳定
某些协议不希望频繁切换后端
```

<span style="color:red">重点：</span>

```text
sh 追求同一个源地址尽量落到固定后端。
```

---

## 13. rr 与 sh 对比

| 对比项 | `rr` | `sh` |
|---|---|---|
| 全称 | Round Robin | Source Hashing |
| 中文 | 轮询 | 源地址哈希 |
| 分配依据 | 后端顺序 | 客户端源 IP |
| 同一客户端是否固定 | 不一定 | 通常更固定 |
| 适合场景 | 均匀分摊 | 源地址亲和 |
| 后端变化影响 | 较小 | 可能导致重新映射 |

<span style="color:red">重点：</span>

```text
rr 看顺序；
sh 看来源。
```

---

## 14. Masq 转发模式

`Masq` 表示：

```text
Masquerading / NAT 转发模式
```

在输出中：

```text
-> 10.244.1.23:18999 Masq
```

表示：

```text
该 Real Server 使用 Masq 转发模式。
```

Masq 模式特点：

| 特点 | 说明 |
|---|---|
| 请求路径 | 经过 IPVS 节点 |
| 回包路径 | 通常也经过 IPVS 节点 |
| 后端是否需要 VIP | 不需要 |
| 网络要求 | 相对简单 |
| 节点压力 | IPVS 节点承担双向流量 |

<span style="color:red">重点：</span>

```text
Masq 模式更容易保证回包路径，但 IPVS 节点承担请求和回包流量。
```

---

## 15. Weight / ActiveConn / InActConn

| 字段 | 含义 | 排查意义 |
|---|---|---|
| `Weight` | 后端权重 | 权重为 0 时通常不接新流量 |
| `ActiveConn` | 活跃连接数 | 看当前是否有活跃连接 |
| `InActConn` | 非活跃连接数 | 短连接 / UDP 流记录常见 |

<span style="color:red">重点：</span>

```text
访问时 ActiveConn / InActConn 完全不变化，可能说明流量没有进入该 Virtual Server。
```

---

## 16. 常用命令

查看虚拟服务和后端：

```bash
ipvsadm -Ln
```

查看连接记录：

```bash
ipvsadm -Lnc
```

查看统计信息：

```bash
ipvsadm -Ln --stats
```

查看速率：

```bash
ipvsadm -Ln --rate
```

按端口快速过滤：

```bash
ipvsadm -Ln | grep <port> -A 5
```

---

## 17. IPVS 与 iptables / conntrack / route 的关系

| 组件 | 作用 |
|---|---|
| `iptables` | 规则入口、标记、部分 NAT/SNAT/MASQUERADE |
| `IPVS` | Virtual Server 到 Real Server 的四层负载均衡 |
| `conntrack` | 记录连接和 NAT 映射状态 |
| `ip route` | 决定包最终怎么到达后端或客户端 |

<span style="color:red">重点：</span>

```text
iptables 更偏规则和辅助处理；
IPVS 更偏内核四层负载均衡；
conntrack 负责连接状态；
ip route 负责实际路由。
```

---

## 18. IPVS 与 iptables DNAT 对比

| 对比项 | iptables DNAT | IPVS |
|---|---|---|
| 核心对象 | rule / chain | Virtual Server / Real Server |
| 核心能力 | 修改目标地址 | 四层负载均衡 |
| 多后端支持 | 需要多条规则配合 | 原生支持 |
| 后端选择 | 靠规则 | 靠 Scheduler |
| 查看方式 | `iptables-save` | `ipvsadm -Ln` |
| 典型场景 | 简单地址转换 | VIP 到多后端负载均衡 |

<span style="color:red">重点：</span>

```text
iptables DNAT 更像“改目标地址”；
IPVS 更像“带后端选择能力的内核负载均衡器”。
```

---

## 19. IPVS 排查顺序

遇到 VIP 转发异常，可按顺序检查：

```text
1. 是否存在对应 Virtual Server
2. 协议是否正确：TCP / UDP
3. VIP 和端口是否正确
4. Scheduler 是否符合预期
5. 是否存在 Real Server
6. Real Server 的 IP 和端口是否正确
7. Forward 是否为预期模式，如 Masq
8. Weight 是否为 0
9. ActiveConn / InActConn 是否变化
10. ipvsadm -Lnc 是否有连接记录
11. tcpdump 是否能抓到请求和回包
12. iptables / conntrack / route 是否符合预期
```

---

## 20. 常见异常判断

| 现象 | 可能原因 |
|---|---|
| 没有 Virtual Server | IPVS 规则未写入 / 被清理 / VIP 端口不匹配 |
| 有 Virtual Server，无 Real Server | 后端未同步 / Pod 未就绪 / Endpoint 为空 |
| Real Server 存在，但连接数不变 | 流量未进入 IPVS / 协议端口不匹配 |
| 后端收到包，客户端无响应 | 回包路径异常 / NAT / conntrack / 路由问题 |
| Weight 为 0 | 后端可能不会接收新流量 |
| rr 下流量不均 | 连接长短差异 / conntrack 记录 / 后端状态影响 |
| sh 下同源固定 | 正常现象，源地址哈希导致 |

---

## 21. 最重要总结

```text
IPVS：Linux 内核四层负载均衡器
Virtual Server：VIP:Port，对外入口
Real Server：PodIP:Port，真实后端
Scheduler：选择后端的算法
rr：轮询，按顺序分配
sh：源地址哈希，同源尽量固定后端
Masq：NAT/Masquerading 模式，请求和回包通常经过 IPVS 节点
ipvsadm -Ln：查看虚拟服务和真实后端
ipvsadm -Lnc：查看连接记录
```

<span style="color:red">一句话记忆：</span>

```text
rr 看顺序，sh 看来源，Masq 看回包路径。
```

<span style="color:red">核心理解：</span>

```text
IPVS = Virtual Server + Real Server + Scheduler + Forward Method。
```
