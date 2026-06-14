# network-agent

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

## 