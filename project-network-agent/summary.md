# Linux network

## 包路径

外部访问本机

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

外部包进入并转发

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

本机进程发包

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

## 五链，nat/mangle表

| 包路径阶段       | 五链          | 包处理时机                     | `nat` 表常见动作      | `mangle` 表常见动作 | 常见用途                           | 排查关注点                             |
| ---------------- | ------------- | ------------------------------ | --------------------- | ------------------- | ---------------------------------- | -------------------------------------- |
| 外部包刚进入机器 | `PREROUTING`  | 路由判断前                     | `DNAT`                | `MARK`              | 外部流量改目标地址；路由前打标记   | 包是否进入本机；DNAT/MARK 计数是否增长 |
| 目标是本机       | `INPUT`       | 路由判断后，交给本机进程前     |                       |                     | 本机服务入站处理                   | 是否是访问本机进程                     |
| 经过本机转发     | `FORWARD`     | 路由判断后，转发过程中         |                       | 可做特殊处理        | 转发流量检查                       | 是否被 filter FORWARD 拦截             |
| 本机进程发包     | `OUTPUT`      | 本机进程产生包后，路由判断前后 | `DNAT`                | `MARK`              | 本机访问目标时改写；本机发包打标记 | 本机访问目标时是否命中 OUTPUT          |
| 包离开机器前     | `POSTROUTING` | 路由判断后，出网卡前           | `SNAT` / `MASQUERADE` |                     | 修改源地址；保证回包路径           | 是否需要 SNAT；回包路径是否正确        |

---

## DNAT / SNAT / MASQUERADE

|             | DNAT                    | SNAT                      | MASQUERADE      |
| ----------- | ----------------------- | ------------------------- | --------------- |
| 修改对象    | 目标 IP / 端口          | 源 IP / 端口              | 源 IP / 端口    |
| 常见表      | `nat`                   | `nat`                     | `nat`           |
| 常见链      | `PREROUTING` / `OUTPUT` | `POSTROUTING`             | `POSTROUTING`   |
| 典型用途    | 转发到新目标            | 保证回包路径 / 隐藏源地址 | 动态出口 NAT    |
| 是否指定 IP | 指定新目标 IP           | 指定新源 IP               | 自动选择出口 IP |

```text
DNAT：改目标，让包去正确的地方
SNAT：改源，让回包回到正确的地方
MASQUERADE：动态 SNAT，自动使用出口网卡 IP
```

## conntrack

`connection tracking` （Linux 内核维护的一张连接状态表）

- DNAT 改请求包的目标地址，conntrack 负责让回包源地址被还原
- SNAT 改请求包的源地址，conntrack 负责让回包目标地址被还原

---

## MARK + ip rule + ip route

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

## IPVS

> Masq 模式（`NAT / Masquerading 转发模式`）下，请求和回包通常都经过 IPVS 节点。

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

| 对比项     | iptables DNAT    | IPVS                         |
| ---------- | ---------------- | ---------------------------- |
| 核心对象   | rule / chain     | Virtual Server / Real Server |
| 核心能力   | 修改目标地址     | 四层负载均衡                 |
| 多后端支持 | 需要多条规则配合 | 原生支持                     |
| 后端选择   | 靠规则           | 靠 Scheduler                 |
| 查看方式   | `iptables-save`  | `ipvsadm -Ln`                |
| 典型场景   | 简单地址转换     | VIP 到多后端负载均衡         |

```text
iptables DNAT 更像“改目标地址”；
IPVS 更像“带后端选择能力的内核负载均衡器”。
```

## ipvsadm -Ln

```text
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.0.0.100:18999 rr
  -> 10.244.1.23:18999            Masq    1      0          5
  -> 10.244.2.45:18999            Masq    1      0          3
```

```
存在一个 TCP 虚拟服务：10.0.0.100:18999
它使用 rr 轮询算法
下面有两个真实后端：
  10.244.1.23:18999
  10.244.2.45:18999
这两个后端都使用 Masq/NAT 转发模式
权重都是 1
当前活跃连接数都是 0
非活跃连接数分别是 5 和 3
```

> **调度算法**：
>
> - `rr = Round Robin` ：轮询
> - `sh = Source Hashing` ：源地址哈希

---

## TEE 流量镜像

```bash
iptables -t mangle -A PREROUTING \
  -d 10.0.0.100 \
  -p udp --dport 18999 \
  -j TEE --gateway 192.168.1.200
```

| 维度          | 原包                                | TEE 镜像包                     | 关键说明                                                     |
| ------------- | ----------------------------------- | ------------------------------ | ------------------------------------------------------------ |
| 本质          | 正常业务包                          | clone 出来的副本               | <span style="color:red">原包是主路径，镜像包是旁路副本</span> |
| 路径          | 继续原规则链、路由、NAT/IPVS        | 发往 `--gateway` 指定下一跳    | 路径不同                                                     |
| 下一跳        | 正常路由决定                        | `--gateway` 指定               | TEE 改的是副本下一跳                                         |
| 源 IP         | 复制时刻的源 IP，后续可能被 SNAT    | 复制时刻的源 IP                | 复制瞬间通常相同                                             |
| 目的 IP       | 复制时刻的目的 IP，后续可能被 DNAT  | 复制时刻的目的 IP              | TEE 不会自动改目的 IP                                        |
| 源端口        | 复制时刻的源端口                    | 复制时刻的源端口               | 通常相同                                                     |
| 目的端口      | 复制时刻的目的端口，后续可能被 DNAT | 复制时刻的目的端口             | <span style="color:red">TEE 不会自动改端口</span>            |
| 协议          | TCP / UDP / ICMP 等                 | 与复制时刻原包一致             | 通常相同                                                     |
| 源 MAC        | 当前链路上的源 MAC                  | 重新发往 gateway 时重新封装    | 通常不同                                                     |
| 目的 MAC      | 当前链路上的目的 MAC                | 通常是 gateway 的 MAC          | <span style="color:red">L2 目的 MAC 通常不同</span>          |
| mark / fwmark | 如果复制前已有 mark，则原包带 mark  | 通常继承复制时刻 skb mark      | <span style="color:red">取决于 TEE 执行前是否已 MARK</span>  |
| conntrack     | 正常参与连接跟踪                    | 不应视为完整业务连接           | 镜像包不是新建完整连接                                       |
| 是否期待回包  | 是                                  | 通常不应期待有效回包           | TEE 更适合单向镜像                                           |
| 常见用途      | 正常业务通信                        | 镜像、抓包、审计、UDP 单向复制 |                                                              |

---

