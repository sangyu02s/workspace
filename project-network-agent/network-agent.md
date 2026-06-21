# network-agent

---

## 集群环境

当前环境是 **VIP 漂移到 master node1 + IPVS Virtual Server 负责四层负载均衡 + Masq/NAT 模式转发 + flannel 负责跨节点 Pod 网络路由**。

```text
集群环境节点信息：k8s-node，node1、node2、node3。
network-agent在 node1/node2/node3 均有实例，但 vip 仅在 node1 或 node2 上进行主备漂移。
业务服务运行在业务节点上，如前台入口 ER 服务31943有两个实例分别运行在 node1/node2上，也有一些具有两个实例的其他服务 pod 或是运行在node2/node3上、或是运行在node1/node2上等等。
```

---

## 转发机制

**node1 作为 VIP 入口节点，通过 IPVS/LVS 的 NAT/Masq 模式，把** **`vip:31943`** **负载均衡到后端 PodIP:31943。Pod 流量跨节点时，通过 flannel overlay 网络转发。**

```text
client -> node1 vip:31943 -> IPVS 选后端 -> pod-ip:31943
```

```text
VIP 绑定在 node1 eth0
  ↓
IPVS Virtual Server: vip:31943
  ↓
IPVS rr 选择 Real Server
  ↓
Real Server: pod-on-node1-ip:31943 / pod-on-node2-ip:31943
  ↓
通过 flannel/cni0 路由到 Pod
```

```text
客户端访问 vip:31943
  ↓
包进入 node1 eth0
  ↓
node1 发现 vip 是本机 eth0 secondary IP
  ↓
IPVS 匹配 TCP vip:31943
  ↓
IPVS rr 选择一个 Real Server
  ↓
改写/转发到某个 pod-ip:31943
  ↓
Linux 查路由
  ↓
如果 Pod 在远端节点：走 flannel.1
如果 Pod 在本节点：走 cni0
  ↓
Pod 收到请求
  ↓
回包按 IPVS/NAT/conntrack/MASQUERADE 关系返回 client
```

---

## 包路径示例

**请求方向**

```text
client-ip:random-port
  ↓
dst = vip:31943
  ↓
node1 eth0 收包
  ↓
因为 VIP 绑定在 node1 eth0，包进入 node1 协议栈
  ↓
IPVS 匹配 Virtual Server：vip:31943
  ↓
IPVS rr 选择一个 Real Server
  ↓
如果选中 172.20.3.73:31943：
    route -> flannel.1 -> 远端 node -> Pod
  ↓
如果选中 172.20.2.226:31943：
    route -> cni0 -> 本机 Pod
```

**如果选中 node2（172.20.3.73）**

```text
client
  ↓
node1 eth0
  ↓
IPVS: vip:31943
  ↓
RealServer: 172.20.3.73:31943
  ↓
ip route get 172.20.3.73
  ↓
via 172.20.3.0 dev flannel.1
  ↓
flannel overlay
  ↓
远端 node
  ↓
Pod 172.20.3.73:31943
```

**如果选中 node1（172.20.2.226）（与 vip 同节点）**

```text
client
  ↓
node1 eth0
  ↓
IPVS: vip:31943
  ↓
RealServer: 172.20.2.226:31943
  ↓
ip route get 172.20.2.226
  ↓
dev cni0
  ↓
本机 Pod 172.20.2.226:31943
```

---

## IPVS Masq

IPVS 使用 NAT/Masquerading 类型的转发模式。你可以先这样理解：

```text
客户端以为自己访问的是 vip:31943
IPVS 把请求转发给 pod-ip:31943
回包时再通过 NAT 关系保证客户端看到的仍然是 vip:31943 这条连接
```

这和普通“路由转发”不一样，它是有状态的四层负载均衡。

所以这套机制通常依赖：

```text
IPVS 连接表
conntrack / NAT 状态
POSTROUTING MASQUERADE/SNAT 规则
路由到 Pod 的能力
```

你可以进一步查：

```bash
ipvsadm -Ln --stats
ipvsadm -Ln --rate
cat /proc/net/ip_vs_conn | grep 31943
conntrack -L | grep 31943
```

看 IPVS 连接实时选中了哪个后端，访问业务时，在 node1 执行：

```bash
ipvsadm -Ln --stats | grep -A10 31943
ipvsadm -Ln --rate | grep -A10 31943
```

或者：

```bash
watch -n 1 "ipvsadm -Ln --stats | grep -A10 31943"
```

看哪个 Real Server 的连接数、包数在增长。