# iptables TEE 核心笔记

> 适合 Linux 流量镜像、旁路抓包、UDP 流量复制、网络代理项目入门使用。  
> 本笔记只保留重点，重点关注：`TEE` 概念、原包与镜像包对比、常见示例、限制和排查方法。

---

## 1. TEE 是什么

`TEE` 是 iptables 的一个 target，作用是：

> **复制一份网络包，并把副本发送到指定 gateway。**

```text
原包：继续走原路径
镜像包：发往 --gateway 指定的下一跳
```

<span style="color:red">重点：</span>

```text
TEE 是“复制包”，不是“改包”。
TEE 不等于 DNAT，也不等于负载均衡。
```

---

## 2. TEE 基本路径图

```text
原始路径：

client
  ↓
Linux 节点
  ↓
真实目标


TEE 后：

client
  ↓
Linux 节点
  ├─ 原包：继续去真实目标
  └─ 镜像包：发给 --gateway 指定机器
```

---

## 3. TEE 常用位置

TEE 通常用在：

```text
mangle 表
```

常见链：

```text
mangle PREROUTING
mangle POSTROUTING
```

常见用途：

```text
流量镜像
旁路抓包
安全审计
日志分析
协议分析
旁路监控
UDP 单向流量复制
```

---

## 4. TEE 基本语法

```bash
iptables -t mangle -A <CHAIN> <匹配条件> \
  -j TEE --gateway <mirror-gateway-ip>
```

示例：

```bash
iptables -t mangle -A PREROUTING \
  -i eth0 \
  -j TEE --gateway 192.168.1.200
```

含义：

```text
从 eth0 进入的包复制一份；
原包继续正常处理；
镜像包发给 192.168.1.200。
```

---

## 5. TEE 核心特点

| 特点 | 说明 |
|---|---|
| 复制对象 | 网络包 packet |
| 原包 | 不受影响，继续原路径 |
| 镜像包 | 发往 `--gateway` 指定地址 |
| 常用表 | `mangle` |
| 常用链 | `PREROUTING` / `POSTROUTING` |
| 是否改 IP/端口 | TEE 本身不改 |
| 是否适合 UDP 镜像 | 相对适合 |
| 是否适合 TCP 业务复制 | 通常不适合 |

<span style="color:red">重点：</span>

```text
TEE 复制的是 packet，不是完整业务请求语义。
```

---

## 6. 原包与镜像包对比

> 这里的“镜像包”指 iptables `TEE` target clone 出来的包副本。

| 维度 | 原包 | TEE 镜像包 | 关键说明 |
|---|---|---|---|
| 本质 | 正常业务包 | clone 出来的副本 | <span style="color:red">原包是主路径，镜像包是旁路副本</span> |
| 路径 | 继续原规则链、路由、NAT/IPVS | 发往 `--gateway` 指定下一跳 | 路径不同 |
| 下一跳 | 正常路由决定 | `--gateway` 指定 | TEE 改的是副本下一跳 |
| 业务目标 | 原逻辑决定，后续可能被 DNAT/IPVS 改写 | 复制时刻包里的目的 IP/端口 | 镜像包不会自动跟随原包后续 DNAT |
| 源 IP | 复制时刻的源 IP，后续可能被 SNAT | 复制时刻的源 IP | 复制瞬间通常相同 |
| 目的 IP | 复制时刻的目的 IP，后续可能被 DNAT | 复制时刻的目的 IP | TEE 不会自动改目的 IP |
| 源端口 | 复制时刻的源端口 | 复制时刻的源端口 | 通常相同 |
| 目的端口 | 复制时刻的目的端口，后续可能被 DNAT | 复制时刻的目的端口 | <span style="color:red">TEE 不会自动改端口</span> |
| 协议 | TCP / UDP / ICMP 等 | 与复制时刻原包一致 | 通常相同 |
| Payload | 原始业务数据 | 与复制时刻原包一致 | 通常相同 |
| 源 MAC | 当前链路上的源 MAC | 重新发往 gateway 时重新封装 | 通常不同 |
| 目的 MAC | 当前链路上的目的 MAC | 通常是 gateway 的 MAC | <span style="color:red">L2 目的 MAC 通常不同</span> |
| TTL | 复制时刻的 TTL，转发中可能递减 | 通常继承复制时刻 TTL，后续也可能递减 | 基本相同但可能变化 |
| Checksum | 当前包 checksum | clone 后通常保持；若后续改写会重算 | 复制瞬间通常一致 |
| mark / fwmark | 如果复制前已有 mark，则原包带 mark | 通常继承复制时刻 skb mark | <span style="color:red">取决于 TEE 执行前是否已 MARK</span> |
| conntrack | 正常参与连接跟踪 | 不应视为完整业务连接 | 镜像包不是新建完整连接 |
| NAT | 原包可能继续 DNAT/SNAT | 复制后走独立路径，不自动跟随原包后续 NAT | 不同 |
| IPVS 调度 | 可能进入 IPVS 被调度 | 通常不参与原 IPVS 调度 | 不同 |
| 是否影响原业务 | 是业务本体 | 通常不影响原业务 | 镜像包用于旁路 |
| 是否期待回包 | 是 | 通常不应期待有效回包 | TEE 更适合单向镜像 |
| TCP 服务能否直接处理 | 正常可以 | 普通 TCP 服务通常不可以 | TCP 状态机不成立 |
| UDP 服务能否直接处理 | 正常可以 | 取决于目的 IP/端口和接收方式 | UDP 更适合但仍需注意投递条件 |
| 是否等于复制业务请求 | 是业务请求本体 | 不是完整业务请求，只是包副本 | <span style="color:red">TEE 复制的是包，不是连接/请求</span> |
| 是否负载均衡 | 取决于原路径 | 否 | TEE 不做后端选择 |
| 常见用途 | 正常业务通信 | 镜像、抓包、审计、UDP 单向复制 | 不同 |

---

## 7. 关于 mark 的重点说明

镜像包是否带 mark，取决于 **TEE 执行前原包是否已经被打上 mark**。

### 7.1 先 MARK，再 TEE

```text
mangle PREROUTING:
  rule 1: MARK --set-mark 0x100
  rule 2: TEE --gateway 192.168.1.200
```

结果：

```text
原包：mark = 0x100
镜像包：通常也带 mark = 0x100
```

---

### 7.2 先 TEE，再 MARK

```text
mangle PREROUTING:
  rule 1: TEE --gateway 192.168.1.200
  rule 2: MARK --set-mark 0x100
```

结果：

```text
镜像包：复制时还没有 mark，通常不带 0x100
原包：后续才被打上 mark = 0x100
```

<span style="color:red">重点：</span>

```text
镜像包是否继承 mark，关键看 TEE 执行时原包 skb 上是否已有 mark。
```

---

## 8. 关于 IP / MAC 的重点说明

很多人会误解 `--gateway`。

```bash
-j TEE --gateway 192.168.1.200
```

它不是说：

```text
把镜像包的目的 IP 改成 192.168.1.200
```

而是说：

```text
把镜像包发往 192.168.1.200 这个下一跳
```

常见情况：

```text
L3 目的 IP：仍然是复制时刻的原目的 IP
L4 目的端口：仍然是复制时刻的原目的端口
L2 目的 MAC：变成 gateway 对应的 MAC
```

<span style="color:red">重点：</span>

```text
TEE 改的是副本的下一跳，不是副本的业务目的 IP/Port。
```

---

## 9. TEE 所在链影响镜像包内容

TEE 复制的是 **命中规则那一刻** 的包。

### 9.1 放在 mangle PREROUTING

```text
外部包进入
  ↓
mangle PREROUTING
  └─ TEE 复制
  ↓
nat PREROUTING
  └─ 可能 DNAT
```

此时镜像包通常是：

```text
DNAT 前的状态
```

---

### 9.2 放在 mangle POSTROUTING

```text
mangle PREROUTING
  ↓
nat PREROUTING：可能 DNAT
  ↓
路由判断
  ↓
FORWARD
  ↓
mangle POSTROUTING
  └─ TEE 复制
  ↓
nat POSTROUTING：可能 SNAT/MASQUERADE
```

此时镜像包通常是：

```text
已经过 DNAT，但可能还没经过最终 SNAT/MASQUERADE 的状态
```

<span style="color:red">重点：</span>

```text
TEE 复制的是当时的包状态，不是固定复制“原始包”或“最终包”。
```

---

## 10. 常见示例

### 10.1 镜像所有入站流量到监控机

```bash
iptables -t mangle -A PREROUTING \
  -i eth0 \
  -j TEE --gateway 192.168.1.200
```

适合：

```text
旁路抓包
流量审计
入站流量分析
```

---

### 10.2 只镜像某个 UDP 端口

```bash
iptables -t mangle -A PREROUTING \
  -i eth0 \
  -p udp --dport 162 \
  -j TEE --gateway 192.168.1.200
```

适合：

```text
SNMP Trap 镜像
UDP 告警流量旁路分析
UDP 日志流量复制
```

---

### 10.3 镜像某个 VIP:Port

```bash
iptables -t mangle -A PREROUTING \
  -d 10.0.0.100 \
  -p udp --dport 18999 \
  -j TEE --gateway 192.168.1.200
```

<span style="color:red">重点：</span>

```text
TEE 不会自动把副本改成另一个服务或另一个端口。
如果副本要改目标 IP/端口，需要额外 DNAT / REDIRECT / 代理 / eBPF 处理。
```

---

### 10.4 同时镜像入站和出站流量

入站：

```bash
iptables -t mangle -A PREROUTING \
  -i eth0 \
  ! -s 192.168.1.200 \
  -j TEE --gateway 192.168.1.200
```

出站：

```bash
iptables -t mangle -A POSTROUTING \
  -o eth0 \
  ! -d 192.168.1.200 \
  -j TEE --gateway 192.168.1.200
```

<span style="color:red">重点：</span>

```text
双向镜像时要避免把镜像流量再次复制，造成循环。
```

---

## 11. TEE 与 DNAT / IPVS 对比

| 对比项 | TEE | DNAT | IPVS |
|---|---|---|---|
| 核心能力 | 复制包 | 修改目标地址 | 四层负载均衡 |
| 是否产生副本 | 是 | 否 | 否 |
| 是否改 IP/端口 | TEE 本身不改 | 会改目标 IP/端口 | 会选择后端并转发 |
| 是否选择后端 | 否 | 否 | 是 |
| 常用表/工具 | `mangle` | `nat` | `ipvsadm` |
| 常见用途 | 镜像、旁路分析 | 转发、端口映射 | VIP 到多后端转发 |

<span style="color:red">重点：</span>

```text
TEE 复制包；
DNAT 改目标；
IPVS 做负载均衡。
```

---

## 12. TEE 的重要限制

| 限制 | 说明 |
|---|---|
| `--gateway` 不是业务目标 | 它是下一跳，不会改 L3 目的 IP |
| 不改端口 | TEE 不会把 `18999` 改成 `18990` |
| 不适合 TCP 完整业务复制 | TCP 状态机、握手、ACK、序列号对不上 |
| 不是负载均衡 | 不支持 `rr`、`sh` 等调度 |
| 会放大流量 | 一份流量复制成两份 |
| 可能产生环路 | 双向镜像时尤其要小心 |
| 镜像包不一定被业务进程收到 | 取决于目的 IP/端口、路由、本机接收方式 |

---

## 13. 适用场景与不适用场景

| 场景 | 是否适合 TEE |
|---|---|
| UDP 单向流量镜像 | 适合 |
| 旁路抓包 | 适合 |
| 安全审计 | 适合 |
| 协议分析 | 适合 |
| 流量观测 | 适合 |
| TCP 完整业务复制 | 不适合 |
| 需要修改副本端口 | TEE 本身不够 |
| 需要负载均衡 | 不适合，用 IPVS 等 |

---

## 14. TEE 排查命令

查看 TEE 规则：

```bash
iptables-save -t mangle | grep TEE
```

查看 mangle 表计数：

```bash
iptables -t mangle -L -n -v --line-numbers
```

查看内核模块：

```bash
lsmod | grep xt_TEE
```

尝试加载模块：

```bash
modprobe xt_TEE
```

抓包验证：

```bash
tcpdump -ni any host <原目标IP> or host <mirror-gateway-ip>
```

在镜像机抓包：

```bash
tcpdump -ni any host <client-ip>
```

---

## 15. TEE 排查顺序

```text
1. TEE 规则是否存在？
2. TEE 规则所在链是否正确？
3. 包是否真的经过该链？
4. 匹配条件是否正确？
5. pkts/bytes 是否增长？
6. xt_TEE 模块是否可用？
7. --gateway 是否可达？
8. 镜像机是否能抓到副本？
9. 是否需要在镜像路径上做 DNAT/REDIRECT？
10. 是否出现重复复制或环路？
```

---

## 16. 最重要总结

```text
TEE：复制一份 packet，并把副本发给指定 gateway。
```

<span style="color:red">最重要：</span>

```text
TEE 复制包，不改包。
TEE 指定的是 gateway，不是新的业务目的 IP。
TEE 复制的是 packet，不是 connection，也不是完整业务请求。
镜像包是否带 mark，取决于 TEE 执行前原包是否已有 mark。
```

<span style="color:red">一句话记忆：</span>

```text
原包走业务主路径；
镜像包走旁路观察路径；
TEE 适合 UDP/无状态流量镜像，不适合直接复制完整 TCP 业务连接。
```
