# IPC 命令

## 概述

* 我们把底层的pub/sub系统做了封装，封装成了一套IPC系统，用于进程间的通讯
* 在IPC系统中，**每个进程用一个名字来标识**，命令的执行进程可以通过名字主动连接到命令的发送进程，接收&执行对方发送过来的命令。例如：网关进程要执行策略进程mm1和策略进程mm2的下单指令，需要主动connect到策略进程mm1和策略进程mm2，才能收到这两个策略进程发送过来的下单命令
    * `/hft/zmq/command/kraken_gateway.ipc`
    * `/hft/zmq/command/mm1.ipc`
    * `/hft/zmq/command/mm2.ipc`

## 架构

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 IPC Command 系统                                     │
│                                                                                     │
│   ┌─────────────────────┐                         ┌─────────────────────┐           │
│   │   进程 A (Gateway)   │                         │   进程 B (Strategy) │           │
│   │                     │                         │                     │           │
│   │ ┌─────────────────┐ │       command           │ ┌─────────────────┐ │           │
│   │ │ CommandPublisher│─┼────────────────────────▶│ │CommandSubscriber│ │           │
│   │ └─────────────────┘ │                         │ └─────────────────┘ │           │
│   │                     │                         │                     │           │
│   │ ┌─────────────────┐ │       command           │ ┌─────────────────┐ │           │
│   │ │CommandSubscriber│◀┼─────────────────────────│ │ CommandPublisher│ │           │
│   │ └─────────────────┘ │                         │ └─────────────────┘ │           │
│   └─────────────────────┘                         └─────────────────────┘           │
│                                                                                     │
│   通信地址：/hft/zmq/command/{process_id}.ipc                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 与发布&订阅系统的区别

| 特性 | IPC PubSub | IPC Command |
|------|------------|-------------|
| 用途 | 市场数据分发 | 控制命令传递 |
| 序列化 | Protobuf | JSON |
| 地址 | `exchange.channel.ipc` | `command/{process_id}.ipc` |
| 订阅方式 | 按交易对过滤 | 按进程ID和命令类型过滤 |
| 数据类型 | Ticker/Trade/Order | 任意命令 |

## 命令的设计

### 命令类型

设计了三种类型的命令，当前版本实现了**单播**和**广播**，可以满足目前的业务需求

* **单播**: `process_id.command_type`，target_id是`process_id`，也就是这条消息应该`process_id`这个进程来处理
    * 例如: `kraken_gateway.start_trading`，kraken_gateway这个进程处理start_trading命令
* **广播**: `*.command_type`，target_id是`*`，所有进程都要处理这条消息
    * 例如: `*.shutdown`，所有进程shutdown
* **组播**(当前版本不支持): `api.*.command_type`, target_id是`api.*`
    * 例如: `api.*.shutdown`，所有api开头的进程shutdown


!!! note "注意"

    * 底层的pub/sub模式是根据**主题(也就是前缀)**来过滤，前缀做如下设计，长度是64个字节:
        * `target_id.command_type`，不足64字节的用空格填充
        * 单播: `process_id.command_type`
        * 广播: `*.command_type`
    * 前缀过滤的细节:
        * 调用`sub_socket.setsockopt(zmq.SUBSCRIBE, b"prefix")`时，内部会检查消息的前缀，只有消息内容（message body）开头和`prefix`匹配的才会投递给SUB
        * 这个`prefix`可以是任意字节序列
        * 如果PUB端发送消息`b"cmd1:payload"`，而SUB设置了`zmq.SUBSCRIBE b"cmd1:"`，那么它会收到；如果是`b"cmd2:payload"`就不会收到
        * 注意：过滤发生在接收端，发送端 PUB 是无感知的
    * 命令设计细节:
        * 用`target_id`来过滤命令
        * 用`command_type`来决定由哪个函数处理目标命令
        * 命令的参数保存在payload中

### 消息格式

```
┌──────────────────────────────┬─────────────────────────────┐
│    Topic (64 bytes 固定)      │      JSON Data (变长)        │
│  {target_id}.{command_type}  │  IPCCommand 序列化数据        │
└──────────────────────────────┴─────────────────────────────┘
```

```
┌──────────────────────────────────────────────┬─────────────────────────────┐
│   target_id.command_type  ─── 64 bytes ───   │       command data          │
└──────────────────────────────────────────────┴─────────────────────────────┘

单播
┌──────────────────────────────────────────────┬─────────────────────────────┐
│   kraken_gateway.start_trading               │       command data          │
└──────────────────────────────────────────────┴─────────────────────────────┘

广播
┌──────────────────────────────────────────────┬─────────────────────────────┐
│   *.shutdown                                 │       command data          │
└──────────────────────────────────────────────┴─────────────────────────────┘
```

### 数据结构

```python
class IPCCommand:
    __slots__ = ('command_id', 'sender_id', 'target_id', 
                 'command_type', 'payload', 'timestamp', '_json_cache')
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `command_id` | str | 命令的唯一标识符，用于命令跟踪和去重：`{process_id}_{timestamp_us}_{counter}` |
| `sender_id` | str | 发送命令的进程标识符 |
| `target_id` | str | 接收命令的目标进程标识符，"*" 表示广播 |
| `command_type` | str | 命令类型，用于区分不同的命令操作，如 `hello`, `ping`, `shutdown` |
| `payload` | Dict | 命令参数，包含命令执行所需的具体数据，是一个json结构 |
| `timestamp` | int | 命令创建时间戳（微秒），可以用来测试命令的延迟 |

## Publisher

**使用方式**

* 每个进程，有一个`IPCCommandSystem`实例，通过`process_id`来标识自己
* 会绑定一个固定的地址：`/hft/zmq/command/{process_id}.ipc`，发送命令的过程就是向这个地址发送消息

```python
# 创建命令系统
system = IPCCommandSystem("gateway")
await system.start()

# 单播命令：发送给特定进程
await system.send_command(
    target_id="strategy_01",
    command_type="start_trading",
    payload={"symbol": "btc/usdt"}
)

# 广播命令：发送给所有订阅者
await system.broadcast(
    command_type="shutdown",
    payload={"reason": "maintenance"}
)

await system.stop()
```

## Subscriber

* 可以订阅多个命令的发布者
    * `command_system.connect_to_process("gateway")`
    * `command_system.connect_to_process("mm")`
* **自动过滤**：只处理发给自己的命令和广播命令
* 每连接到一个目标地址，会启动一个`_receive_worker`协程接收数据，收到的命令放到`command_queue`队列中
* 有一个独立的协程`_process_worker`负责执行这些命令

**协程架构**

```
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │ _receive_worker │  │ _receive_worker │  │ _receive_worker │
  │   (进程 A)       │  │   (进程 B)       │  │   (进程 C)       │
  │                 │  │                 │  │                 │
  │ await recv()    │  │ await recv()    │  │ await recv()    │
  │      ↓          │  │      ↓          │  │      ↓          │
  │ 解析 JSON       │  │ 解析 JSON        │  │ 解析 JSON        │
  │      ↓          │  │      ↓          │  │      ↓          │
  │ put_nowait()    │  │ put_nowait()    │  │ put_nowait()    │
  └───────┬─────────┘  └───────┬─────────┘  └───────┬─────────┘
          │                    │                    │
          └────────────────────┼────────────────────┘
                               ▼
                     ┌─────────────────┐
                     │ _command_queue  │
                     │  asyncio.Queue  │
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │ _process_worker │
                     │                 │
                     │ await get()     │
                     │      ↓          │
                     │ handler(cmd)    │  ← 根据 command_type 分发
                     └─────────────────┘
```

**使用方式**

```python
import asyncio
from hftpy.ipc import IPCCommandSystem, IPCCommand


class StrategyService:
    
    def __init__(self):
        self.command_system = IPCCommandSystem("strategy_01")
    
    async def start(self):
        # 注册命令处理器
        self.command_system.register_handler("start_trading", self.handle_start_trading)
        self.command_system.register_handler("stop_trading", self.handle_stop_trading)
        
        # 连接到其他进程
        await self.command_system.connect_to_process("gateway")
        
        # 启动命令系统
        await self.command_system.start()
    
    async def stop(self):
        await self.command_system.stop()
    
    def handle_start_trading(self, command: IPCCommand):
        print(f"收到启动命令: {command.payload}")
    
    def handle_stop_trading(self, command: IPCCommand):
        print(f"收到停止命令: {command.payload}")


async def main():
    service = StrategyService()
    await service.start()
    
    # 保持运行，等待接收命令
    try:
        while True:
            await asyncio.sleep(1)
    except KeyboardInterrupt:
        await service.stop()


if __name__ == "__main__":
    asyncio.run(main())
```
