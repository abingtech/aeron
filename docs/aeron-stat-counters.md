# AeronStat 监控工具与计数器解读

## 基本原理

`AeronStat` 通过 mmap 读取 Media Driver 的 **CnC（Command and Control）文件**来获取运行时计数器，不需要建立 Aeron 客户端连接。

### 默认监控哪个 Media Driver？

通过 `aeron.dir` 系统属性确定 CnC 文件位置：

```java
// CommonContext.java
new File(getProperty("aeron.dir", AERON_DIR_PROP_DEFAULT), "cnc.dat")
```

默认指向当前用户的临时目录。如果 Pub 和 Sub 各用嵌入式 Media Driver（`launchEmbedded()`），它们各自有独立的 CnC 文件，`AeronStat` 默认只能看其中一个。指定不同的 `-Daeron.dir` 可以切换：

```bash
# 看 Pub 侧
java -Daeron.dir=/var/folders/.../aeron-huangqibing-391204ac-... -cp aeron-samples.jar io.aeron.samples.AeronStat

# 看 Sub 侧
java -Daeron.dir=/var/folders/.../aeron-huangqibing-6e05e02e-... -cp aeron-samples.jar io.aeron.samples.AeronStat
```

## 计数器分类

### 系统级指标（type=0）

| 计数器 | 含义 |
|-------|------|
| Bytes sent / Bytes received | 网卡收发字节总数（含协议开销） |
| Heartbeats sent / received | 心跳帧收发次数 |
| Status Messages sent / received | 流控状态消息收发次数 |
| NAKs sent / received | 重传请求次数（=0 表示无丢包） |
| Short sends | 不完整发送次数 |
| Sender flow control limits | 背压事件次数 |
| Errors | 错误次数 |
| Conductor/Sender/Receiver max cycle time | 各线程最长循环耗时（正常应在微秒级） |
| Bytes currently mapped | 当前 mmap 映射的内存大小 |

### 通道状态（type=6, 7）

| 计数器 | 含义 |
|-------|------|
| snd-channel | 发送通道状态（1=已连接） |
| rcv-channel | 接收通道状态（1=已连接） |

### 位置计数器（type=1~5, 9~11）

| 计数器 | 含义 |
|-------|------|
| pub-pos | Publisher 写入位置 |
| pub-lmt | Publisher 可写入上限（term 边界） |
| snd-pos | Sender 发送位置 |
| snd-lmt | Sender 可发送上限（受接收端流控窗口限制） |
| snd-bpe | 发送端背压事件次数 |
| sub-pos | Subscriber 消费位置 |
| rcv-pos | Receiver 接收位置 |
| rcv-hwm | Receiver 高水位（最大连续接收位置） |
| rcv-naks-sent | 接收端发送的 NAK 次数 |

### 位置计数器关系

```
pub-pos ≤ snd-pos  → 生产者写入，发送者读取发送
snd-pos ≤ snd-lmt  → 发送不能超过接收端流控窗口
snd-lmt ≪ pub-lmt  → 流控窗口远小于 term 总容量（正常）
rcv-pos = rcv-hwm  → 无空洞、无丢包（等于号）
sub-pos = rcv-pos  → Subscriber 消费跟上接收速度

三者全等 = 端到端完美运行，零丢包零积压
```

## 心跳帧与 Status Message 的方向

心跳帧是单向的——只有 Sender（Pub 侧 Media Driver）发送，Receiver（Sub 侧 Media Driver）接收：

```
Pub 侧 Media Driver                    Sub 侧 Media Driver
┌──────────────────┐                  ┌──────────────────┐
│  Sender          │ ── 数据帧 ────►  │  Receiver        │
│                  │ ── 心跳帧 ────►  │                  │
│                  │ ◄── SM(状态消息)─ │                  │
│                  │ ◄── NAK(重传请求) │                  │
└──────────────────┘                  └──────────────────┘
```

| 帧类型 | 方向 | 作用 |
|-------|------|------|
| 数据帧 | Pub→Sub | 传输消息 |
| 心跳帧 | Pub→Sub | 无数据时保持连接（`PUBLICATION_HEARTBEAT_TIMEOUT_NS` 超时触发） |
| Status Message | Sub→Pub | 流控反馈（接收进度+窗口大小），兼起到 keep-alive 作用 |
| NAK | Sub→Pub | 请求重传丢失的数据包 |

如果只有单向 Pub→Sub 通信：
- Pub 侧：Heartbeats sent > 0, Heartbeats received = 0（正常！）
- Sub 侧：Heartbeats sent = 0, Heartbeats received > 0（正常！）

Sub 侧不发送心跳帧（因为没有反向 Publication），但通过 Status Messages 让 Pub 侧知道它还活着。

### 关键源码

- `NetworkPublication.heartbeatMessageCheck()` — Sender 超时无数据时发送心跳帧
- `PublicationImage.insertPacket()` — Receiver 判断 `isHeartbeat` 并更新连接时间戳
