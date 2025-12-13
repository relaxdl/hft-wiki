# IPC 命令

## 概述

* 我们把底层的pub/sub系统做了封装，封装成了一套IPC系统，用于进程间的通讯，是一套简化的IPC系统，能满足自己的业务需求
* **进程标识**：在IPC系统中，**每个进程用一个名字来标识**
* **命令的发送方**：发送命令只需要将命令pub到自己名字对应的地址，不需要关心谁去执行
* **命令的执行方**：通过名字，主动connect到命令的发送方的地址，才能收到对方的命令并执行
    * 例如：网关进程要执行策略进程mm1和策略进程mm2的下单指令，需要主动connect到策略进程mm1和策略进程mm2，才能收到这两个策略进程发送过来的下单命令
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

**发送方决定命令发送给谁（target_id来标识命令的接收方）**，设计了三种类型的命令，当前版本实现了**单播**和**广播**，可以满足目前的业务需求

* **单播**: `process_id.command_type`，target_id是`process_id`，也就是这条消息应该`process_id`这个进程来处理
    * 例如: `kraken_gateway.start_trading`，kraken_gateway这个进程处理start_trading命令
* **广播**: `*.command_type`，target_id是`*`，所有进程都要处理这条消息
    * 例如: `*.shutdown`，所有进程shutdown
* **组播**(当前版本不支持): `api.*.command_type`, target_id是`api.*`
    * 例如: `api.*.shutdown`，所有api开头的进程shutdown


!!! note "注意"

    * 底层的pub/sub模式是根据**主题(也就是前缀)**来过滤，前缀做如下设计，长度是64个字节:
        * `target_id.command_type`，不足64字节的用空格填充，**这种设计方式是为了方便命令的前缀过滤**
        * 单播: `process_id.command_type`，process_id这个进程负责处理这条命令
        * 广播: `*.command_type`，所有进程都会处理这条命令
    * 前缀过滤的细节:
        * 调用`sub_socket.setsockopt(zmq.SUBSCRIBE, b"prefix")`时，内部会检查消息的前缀，只有消息内容（message body）开头和`prefix`匹配的才会投递给SUB
        * 这个`prefix`可以是任意字节序列
        * 如果PUB端发送消息`b"cmd1:payload"`，而SUB设置了`zmq.SUBSCRIBE b"cmd1:"`，那么它会收到；如果是`b"cmd2:payload"`就不会收到
        * 注意：**过滤发生在接收端，发送端 PUB 是无感知的**
    * 命令设计细节:
        * 用`target_id`来过滤命令
        * 用`command_type`来决定由哪个函数处理目标命令
        * 命令的参数保存在payload中

**过滤的本质：subscriber会自动订阅两种 topic**

```python
# 单播：只接收发给自己的命令
subscriber.setsockopt(zmq.SUBSCRIBE, f"{self.process_id}.".encode())  
# 例如：strategy_01.start_trading

# 广播：接收所有广播命令
subscriber.setsockopt(zmq.SUBSCRIBE, b"*.")  
# 例如：*.shutdown
```

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

* 下面的数据结构序列化之后，会增加一个64 bytes的topic，发送出去

```python
class IPCCommand:
    __slots__ = ('command_id', 'sender_id', 'target_id', 
                 'command_type', 'payload', 'timestamp', '_json_cache')
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `command_id` | str | 命令的唯一标识符，用于命令跟踪和去重：`{sender_id}_{timestamp_us}_{counter}` |
| `sender_id` | str | 发送命令的进程标识符 |
| `target_id` | str | 接收命令的目标进程标识符，"*" 表示广播 |
| `command_type` | str | 命令类型，用于区分不同的命令操作，如 `hello`, `ping`, `shutdown` |
| `payload` | Dict | 命令参数，包含命令执行所需的具体数据，是一个json结构 |
| `timestamp` | int | 命令创建时间戳（微秒），可以用来测试命令的延迟 |

一个简单的心跳包的例子：

```json
{
    "command_id": "gateway_1702345678000000_1",
    "sender_id": "gateway",
    "target_id": "*",
    "command_type": "heartbeat",
    "timestamp": 1702345678000000,
    "payload": {
        "status": "running"
    }
}
```

## API

* 整个系统设计好之后，对外暴露的最核心的API就这几个

```python
# 启动系统【发布者，订阅者都需要调用】
command_system = IPCCommandSystem(my_process_id)
command_system.start()

# 发送命令【发布者调用】
send_command(target_id: str, command_type: str, payload: Dict[str, Any]) -> str
broadcast(command_type: str, payload: Dict[str, Any]) -> str

# 连接到远程进程【订阅者需要调用】
connect_to_process(remote_process_id: str)

# 本地注册回调, 处理命令【订阅者调用】
register_handler(command_type: str, handler: Callable[[IPCCommand], None])
```

## Publisher

* 每个进程，有一个`IPCCommandSystem`实例，通过`process_id`来标识自己
* 会绑定一个固定的地址：`/hft/zmq/command/{process_id}.ipc`，**发送命令的过程就是向这个地址pub命令**，不需要关心谁去执行这条命令
* **是否需要独立的协程来发布命令**：不需要

**API**

```python
start()
send_command(target_id: str, command_type: str, payload: Dict[str, Any]) -> str
broadcast(command_type: str, payload: Dict[str, Any]) -> str
```

**使用方式**

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

* 可以通过名字连接到多个地址，订阅多个命令发布者发送的命令
    * `command_system.connect_to_process("gateway")`
    * `command_system.connect_to_process("mm")`
* **自动过滤**：只处理发给自己的命令和广播命令
* **Handler 注册**：按 command_type 注册回调
* **异步并发**：每连接到一个目标地址，会启动一个`_receive_worker`协程接收数据，收到的命令放到一个全局共享的`command_queue`队列中。有一个独立的协程`_process_worker`负责执行这些命令
    * 注意：所有的命令共享一个消息队列和一个消费者协程。这和pub/sub市场数据的设计是不同的。原因是市场数据的推送频率太高了，所以每种类型的消息有一个专门的消息队列和专门的协程负责消费&处理。在RPC这个场景里面，我们内部RPC命令的频率并不高，所以所有的命令共享一个消息队列和消费者协程，简化设计

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

**API**

```python
start()
connect_to_process(remote_process_id: str)
register_handler(command_type: str, handler: Callable[[IPCCommand], None])
```

**使用方式**

```python hl_lines="12-13 16 24-30"
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
    
    # 自定义的回调函数
    def handle_start_trading(self, command: IPCCommand):
        print(f"收到启动命令: {command.payload}")
    
    # 自定义的回调函数
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

## 和Service的集成

### Publisher

**配置信息：**

* ipc_command_publisher: 自己的名字，向这个地址发送命令即可
* ipc_command_subscribers: 远程进程的名字列表，框架会自动connect到这些远程进程，接收这些进程发送来的命令

```json hl_lines="6-7"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "ipc_command_publisher": "gateway",
            "ipc_command_subscribers": ["mm1", "mm2"],
            ...
        }
    }
}
```

**案例：**

* 对于命令的发布者来说，service内部有ipc_command这个内置变量可以直接使用

```python
self.ipc_command: IPCCommandSystem = IPCCommandSystem(ipc_command_publisher) 

# 单播命令：发送给特定进程
await self.ipc_command.send_command(
    target_id="strategy_01",
    command_type="start_trading",
    payload={"symbol": "btc/usdt"}
)

# 广播命令：发送给所有订阅者
await self.ipc_command.broadcast(
    command_type="shutdown",
    payload={"reason": "maintenance"}
)

```

### Subscriber

**配置信息：**

* ipc_command_publisher: 自己的名字，向这个地址发送命令即可
* ipc_command_subscribers: 远程进程的名字列表，框架会自动connect到这些远程进程，接收这些进程发送来的命令


```json hl_lines="6-7"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "ipc_command_publisher": "gateway",
            "ipc_command_subscribers": ["mm1", "mm2"],
            ...
        }
    }
}
```

**案例：**

* 对于订阅者来说，底层框架会自动connect到配置中间中配置的远程进程，接收发送过来的命令，这些底层的细节上层逻辑不需要关系
* 订阅者只需要**注册自定义回调函数，在回调函数中处理远程的请求就可以了**

```python hl_lines="5-6 9-15"
class MMDemoService(ServiceBase):
    
    def on_setup(self) -> bool:
        # 注册命令处理器
        self.ipc_command.register_handler("start_trading", self.on_start_trading)
        self.ipc_command.register_handler("stop_trading", self.on_stop_trading)
        return True
    
    async def on_start_trading(self, command: IPCCommand):
        """处理 start_trading 命令"""
        logger.info(f"收到启动命令: {command.payload}")
    
    async def on_stop_trading(self, command: IPCCommand):
        """处理 stop_trading 命令"""
        logger.info(f"收到停止命令: {command.payload}")
```

## Ping/Pong 案例

一个最简单的双向通信案例：进程 A 发送 ping，进程 B 收到后回复 pong。

### 通信流程

```
┌─────────────┐                      ┌─────────────┐
│  Process A  │                      │  Process B  │
│             │                      │             │
│  send ping ─┼─────── ping ────────▶│  on_ping    │
│             │                      │      │      │
│   on_pong ◀─┼─────── pong ─────────│◀─────┘      │
│             │                      │             │
└─────────────┘                      └─────────────┘
```

### 进程 A（发送 ping，接收 pong）

**配置文件**

```json hl_lines="6-7"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "ipc_command_publisher": "process_a",
            "ipc_command_subscribers": ["process_b"],
            ...
        }
    }
}
```

**示例代码**

```python
class ProcessAService(ServiceBase):
    
    def on_setup(self) -> bool:
        # 注册 pong 处理器
        self.ipc_command.register_handler("pong", self.on_pong)
        return True
    
    def task_send_ping(self):
        """定时任务：发送 ping"""
        self.run_once(
            self.ipc_command.send_command,
            "process_b",  # 目标进程
            "ping",       # 命令类型
            {"time": datetime.now().isoformat()}
        )
    
    async def on_pong(self, command: IPCCommand):
        """收到 pong 回复"""
        logger.info(f"收到 pong: {command.payload}")
```

### 进程 B（接收 ping，回复 pong）

**配置文件**

```json hl_lines="6-7"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "ipc_command_publisher": "process_b",
            "ipc_command_subscribers": ["process_a"],
            ...
        }
    }
}
```

**示例代码**

```python
class ProcessBService(ServiceBase):
    
    def on_setup(self) -> bool:
        # 注册 ping 处理器
        self.ipc_command.register_handler("ping", self.on_ping)
        return True
    
    async def on_ping(self, command: IPCCommand):
        """收到 ping，回复 pong"""
        logger.info(f"收到 ping: {command.payload}")
        
        # 回复 pong
        await self.ipc_command.send_command(
            command.sender_id,  # 回复给发送者
            "pong",
            {"original_time": command.payload.get("time")}
        )
```
