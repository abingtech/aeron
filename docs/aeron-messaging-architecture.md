# Aeron 消息传递架构

Aeron 使用**两份独立的 mmap 文件**实现生产者和消费者之间的零拷贝通信。

## 整体数据流

```
发送端 (同一台机器)
┌──────────────────────────────────────────────┐
│                                              │
│   Pub (客户端进程)           Sender (驱动进程) │
│   ┌─────────────┐          ┌──────────────┐  │
│   │ offer()     │          │ doSend()     │  │
│   │ putBytes()  │  共享    │ 读取         │  │
│   │     ↓       │◄────────►│     ↓        │  │
│   │  ①写入      │ mmap #1  │  ②读取       │  │
│   └─────────────┘          └──────┬───────┘  │
│                                  │          │
└──────────────────────────────────┼──────────┘
                                   │  ③ UDP send
                                   │  DatagramChannel.send()
                                   │
                                   ▼  (网络)

接收端 (另一台机器)
                                   │
┌──────────────────────────────────┼──────────┐
│                                  ▼          │
│   Receiver (驱动进程)            Sub (客户端进程) │
│   ┌──────────────┐          ┌─────────────┐  │
│   │ insertPacket │          │ poll()      │  │
│   │ ③ UDP receive │         │ 读取        │  │
│   │     ↓        │  共享    │     ↑       │  │
│   │  ④写入       │◄────────►│  ⑤读取      │  │
│   └──────────────┘  mmap #2 └─────────────┘  │
│                                              │
└──────────────────────────────────────────────┘
```

## 发送端流程

1. **Publication.offer()** 直接 `putBytes()` 写入 mmap #1 (Publication Log)
2. **Sender** 驱动从同一个 mmap #1 读取数据
3. **Sender** 通过 `DatagramChannel.send()` 走 UDP 发送到网络

→ **Pub 和 Sender 在同一台机器上共享 mmap，零拷贝。**

## 接收端流程

4. **Receiver** 驱动通过 `DatagramChannel.receive()` 从网络接收 UDP 数据报
5. **Receiver** 调用 `TermRebuilder.insert()` 将数据 `putBytes()` 写入 mmap #2 (Image Log)
6. **Sub** 通过 `Image.poll()` 从同一个 mmap #2 读取数据

→ **Receiver 和 Sub 在同一台机器上共享 mmap，零拷贝。**

## 关键设计

- 唯一的**数据拷贝**发生在 UDP send/receive 这一层 (网络 I/O)
- 其余环节（Pub→Sender、Receiver→Sub）都是共享内存的指针/偏移量操作
- 通过 volatile/ordered 语义保证跨进程内存可见性
- IPC 场景下连网络层都跳过，纯 mmap 通信

## 核心文件

- `aeron-client/.../LogBuffers.java` — 客户端 mmap 封装 (第84行 `fileChannel.map`)
- `aeron-client/.../ConcurrentPublication.java` — `offer()` 写入 termBuffer
- `aeron-driver/.../Sender.java` — 驱动发送代理
- `aeron-driver/.../NetworkPublication.java` — `sendData()` 从 mmap 读取并调用 UDP send
- `aeron-driver/.../Receiver.java` — 驱动接收代理
- `aeron-driver/.../PublicationImage.java` — `insertPacket()` 写入 mmap #2
- `aeron-client/.../Image.java` — `poll()` 从 mmap #2 读取
- `aeron-driver/.../buffer/MappedRawLog.java` — 驱动端 mmap 封装
