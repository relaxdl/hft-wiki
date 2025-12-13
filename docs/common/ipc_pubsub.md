# IPC 发布&订阅

## 概述

对于交易所的消息，策略进程可以直接从交易所订阅，消息经过解析后，会分发到对应的`on_<type>`回调中处理。这在简单的业务场景是没有问题的。对于多进程的业务场景，不同的子策略会跑在不同的进程中，我们希望**只有一个进程从交易所订阅一份数据，然后将处理好的数据分发给不同的策略进程**。最常见的两个业务场景是hft market接收处理ticker stream和trade stream，将处理好的ticker和trade分发出去，gateway接收处理order stream，将处理好的order分发出去。**对于消息的订阅者来说，不需要区分消息到底是直接从交易所订阅的，还是从内部的IPC Channel订阅的，在行为上应该是完全一致的**。

* 目前支持`ticker, trade, my_order, my_order_batch`这样一些常见消息的发布和订阅
* 一种类型的消息会发布到一个固定的地址，**发布消息就是向一个地址pub message，一个地址只能有一个发布者，可以有多个订阅者**，对于trade，ticker这类消息，发布者通常是hft market，对于order这类消息，发布者通常是gateway，地址如下：
    * `/hft/zmq/binance.ticker.ipc`
    * `/hft/zmq/binance.trade.ipc`
    * `/hft/zmq/binance.my_order.ipc`
    * `/hft/zmq/binance.my_order_batch.ipc`
* 消息通过protobuf来跨语言传输，对于订阅者来说，**订阅市场直接推送的消息和订阅gateway或者hft market推送的消息行为上是一致的，都是在`on_<type>`回调中处理**
* C++和Python都可以进行消息的订阅&发布

## 消息格式

* 16个字节的topic，可以用于交易对的过滤

```
┌──────────────────────┬────────────────────────┐
│  Topic (16 bytes)    │  Protobuf Data (变长)   │
│  固定长度，右填充空格   │  序列化后的消息数据       │
└──────────────────────┴────────────────────────┘
```

## Publisher

这里介绍的是Python一侧的Publisher的设计

* **单例模式**：全局只有一个实例。方便socket的全局管理
* **懒加载**：Socket只在首次使用的时候创建，不需要调用者显示创建，系统底层会根据消息的类型自动创建对应的Socket
* **多通道的支持**：每个exchange + channel的组合对应一个底层的Socket
* **是否需要独立的协程来发布消息**：不需要

**使用示例**

```python hl_lines="4-15"
from hftpy.ipc import IPCPublisher
from hftpy.common import Exchange, CurrencyPair, Side, OrderStatus, OrderType

# 获取单例实例
publisher = IPCPublisher()

# 发送 Ticker 数据
await publisher.send_ticker(
    exchange=Exchange.BINANCE,
    symbol=CurrencyPair.BTC_USDT,
    ask_price=50000.5,
    ask_volume=1.5,
    bid_price=50000.0,
    bid_volume=2.0
)

# 发送 Trade 数据
await publisher.send_trade(
    exchange=Exchange.BINANCE,
    symbol=CurrencyPair.BTC_USDT,
    price=50000.2,
    volume=0.5,
    side=Side.BUY
)

# 发送订单数据
await publisher.send_my_order(
    exchange=Exchange.KRAKEN,
    symbol=CurrencyPair.BTC_USDT,
    order_id="12345",
    client_order_id="my_order_001",
    side=Side.BUY,
    status=OrderStatus.FILLED,
    type=OrderType.LIMIT,
    price=50000.0,
    avg_price=49999.5,
    volume=0.1,
    filled_volume=0.1,
    fee=2.5,
    extra_data="{}"
)

# 关闭连接
await publisher.close()
```

## Subscriber

这里介绍的是Python一侧的Subscriber的设计

* **观察者模式**：通过`bind_observer()`绑定多个观察者，通常service就是一个观察者
* **异步并发**：每个订阅通道对应一种类型的消息，需要2个协程和一个队列，详细的协程设计图在下方
    * 一个独立协程接收数据【Subscriber】
    * 一个消息队列【Observer】
    * 一个独立的协程消费数据【Observer】
* **自动分发**：解析后自动推送到 Observer 队列，Observer的**最佳实践是对于每种类型的消息，启动一个独立的协程消费处理**

**流程图**

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              IPCSubscriber (生产者协程)                                  │
│                                                                                        │
│   ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐          │
│   │  协程 1              │   │  协程 2              │   │  协程 3             │          │
│   │  BINANCE/TICKER     │   │  BINANCE/TRADE      │   │  KRAKEN/MY_ORDER    │          │
│   │                     │   │                     │   │                     │          │
│   │  await recv()       │   │  await recv()       │   │  await recv()       │          │
│   │       ↓             │   │       ↓             │   │       ↓             │          │
│   │  put_nowait()       │   │  put_nowait()       │   │  put_nowait()       │          │
│   └──────────┬──────────┘   └──────────┬──────────┘   └──────────┬──────────┘          │
│              │                         │                         │                     │
└──────────────┼─────────────────────────┼─────────────────────────┼─────────────────────┘
               │                         │                         │
               ▼                         ▼                         ▼
        ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
        │ticker_queue │           │ trade_queue │           │my_order_queue│
        │  asyncio.Q  │           │  asyncio.Q  │           │  asyncio.Q  │
        └──────┬──────┘           └──────┬──────┘           └──────┬──────┘
               │                         │                         │
               ▼                         ▼                         ▼
┌──────────────┼─────────────────────────┼─────────────────────────┼─────────────────────┐
│              │                         │                         │                     │
│   ┌──────────┴──────────┐   ┌──────────┴──────────┐   ┌──────────┴──────────┐          │
│   │  协程 A              │   │  协程 B              │   │  协程 C             │          │
│   │  process_tickers()  │   │  process_trades()   │   │  process_orders()   │          │
│   │                     │   │                     │   │                     │          │
│   │  await queue.get()  │   │  await queue.get()  │   │  await queue.get()  │          │
│   │       ↓             │   │       ↓             │   │       ↓             │          │
│   │  handle(ticker)     │   │  handle(trade)      │   │  handle(order)      │          │
│   └─────────────────────┘   └─────────────────────┘   └─────────────────────┘          │
│                                                                                        │
│                               DataHandler (消费者协程)                                   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**订阅配置格式**

```python
subscriptions = [
    # (交易所, 频道, Topic列表)
    (Exchange.BINANCE, Channel.TICKER, ["btcusdt", "ethusdt"]),  # 订阅多个 topic
    (Exchange.BINANCE, Channel.TRADE, ""),                       # 订阅所有 topic
    (Exchange.KRAKEN, Channel.MY_ORDER, ["btcusdt"]),           # 订阅单个 topic
]
```

**Observer 接口规范**

* 观察者需要实现相应的队列属性

```python
class MyObserver:
    def __init__(self):
        self.ticker_queue = asyncio.Queue()        # Channel.TICKER
        self.trade_queue = asyncio.Queue()         # Channel.TRADE
        self.my_order_queue = asyncio.Queue()      # Channel.MY_ORDER
        self.my_order_batch_queue = asyncio.Queue() # Channel.MY_ORDER_BATCH
```

**使用示例**

```python hl_lines="35-37"
import asyncio
from hftpy.ipc import IPCSubscriber
from hftpy.common import Exchange, Channel

class DataHandler:
    """数据处理观察者"""
    def __init__(self):
        self.ticker_queue = asyncio.Queue()
        self.trade_queue = asyncio.Queue()
        self.my_order_queue = asyncio.Queue()
        self.my_order_batch_queue = asyncio.Queue()

    async def process_tickers(self):
        """处理 Ticker 数据"""
        while True:
            ticker = await self.ticker_queue.get()
            print(f"收到 Ticker: {ticker.bid_price} / {ticker.ask_price}")

    async def process_trades(self):
        """处理 Trade 数据"""
        while True:
            trade = await self.trade_queue.get()
            print(f"收到 Trade: {trade.price} x {trade.volume}")


async def main():
    # 定义订阅配置
    subscriptions = [
        (Exchange.BINANCE, Channel.TICKER, ["btcusdt", "ethusdt"]),
        (Exchange.BINANCE, Channel.TRADE, ""),  # 订阅所有 trade
        (Exchange.KRAKEN, Channel.MY_ORDER, ""),
    ]

    # 创建订阅者和观察者
    subscriber = IPCSubscriber(subscriptions)
    handler = DataHandler()
    subscriber.bind_observer(handler)

    # 启动所有任务
    await asyncio.gather(
        subscriber.start(),
        handler.process_tickers(),
        handler.process_trades(),
    )


asyncio.run(main())
```

## 和Service的集成

### Publisher

* 对于发布者来说，service内部有`ipc_publisher`这个内置变量可以直接使用

```python
# service内置变量，不需要额外初始化，直接使用
self.ipc_publisher: IPCPublisher = IPCPublisher()

# 发送 Trade 数据
await self.ipc_publisher.send_trade(
    exchange=Exchange.BINANCE,
    symbol=CurrencyPair.BTC_USDT,
    price=50000.2,
    volume=0.5,
    side=Side.BUY
)
```

### Subscriber

* 对于订阅者来说，在service配置中增加如下配置，消息会自动路由到对应的`on_<type>`回调中
    * service本身就是一个Observer，实现了Subscriber Observer要求的队列属性
    * 在启动的时候，service底层会自动将配置信息转换成订阅参数，进行不同channel的订阅
* 下面这个例子中，`ticker，trade，my_order`中配置的symbol都可以是空列表，则不进行过滤，会订阅这个channel所有交易对的消息


**配置信息：**

```json hl_lines="9 17 25 33"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "ipc_subscribers": [
                {
                    "exchange": "binance",
                    "channel": "ticker",
                    "symbols": [
                        "btc/usdt",
                        "eth/usdt"
                    ]
                },
                {
                    "exchange": "binance",
                    "channel": "trade",
                    "symbols": [
                        "btc/usdt",
                        "eth/usdt"
                    ]
                },
                {
                    "exchange": "binance",
                    "channel": "my_order",
                    "symbols": [
                        "btc/usdt",
                        "eth/usdt"
                    ]
                },
                {
                    "exchange": "binance",
                    "channel": "my_order_batch",
                    "symbols": []
                }
            ],
            ...
        }
    }
}
```

**回调函数：**


```python
class MMDemoService(ServiceBase):
    def on_ticker(self, ticker: Any) -> None:
        pass
    def on_trade(self, trade: Any) -> None:
        pass
    def on_my_order(self, my_order: Any) -> None:
        pass
    def on_my_order_batch(self, my_order_list: List[Any]) -> None:
        pass
```