#Websocket Stream

## 设计思路

* Stream是交易所Websocket数据的抽象，底层对应多个TCP链接，通常一个Channel对应一个TCP链接
* **工厂模式 + 观察者模式**：Stream通过`get_exchange_stream()`统一创建，负责**接收交易所推送的消息**，Service是Observer，负责**处理解析后的消息**
    * Stream: 每个交易所，每个Channel的消息都会启动一个协程来接收，接收到的数据通过队列异步分发给多个观察者
    * Service: 每个交易所，每个Channel的消息都会启动一个协程来处理
* 一个Stream可以绑定多个Observer，每个Observer就是一个Service，Stream收到消息后将消息推送到Service对应的queue中，**通过queue实现消息的传递**。消息的接收和消息的处理在不同的协程中，消息的处理不会block消息的接收
* Stream有两个核心变量：`currency_pairs`和`subscribe_channels`
    * `currency_pairs`：订阅的交易对列表，为了简化设计，是所有Channel共享的
    * `subscribe_channels`：订阅的Channel列表
* `start()`函数会启动所有协程，**每个channel对应一个协程**，如果我们要订阅3个Channel，会启动3个协程
* Stream底层封装好了TCP超时，重连的策略&逻辑，每个交易所只需要使用默认的配置即可。对于同一个交易所，不同的Channel共享一个重连策略，这也是合理的，如果一个Channel可以连接上，其它同所的channel应该都可以连接上

## 类的继承结构

```
StreamBase (基类 - ABC)
    ├── KrakenStream
    ├── BinanceStream
    ├── BinanceFutureStream
    ├── CoinbaseStream
    ├── OkxStream
    ├── OkxSwapStream
    ├── KrakenFutureStream
    └── IBStream
```

## 流程图

* **生产者**：`StreamBase`，每个Channel一个协程，从 WebSocket 接收数据
* **解析**：`on_xxx()`，将 JSON 解析为 Ticker/Trade/MyOrder 等对象
* **队列**：`asyncio.Queue`，异步队列，解耦生产者和消费者
* **消费者**：`ServiceBase`，每个队列一个协程，处理业务逻辑

```
┌───────────────────────────────────────────────────────────────┐
│                   StreamBase（生产者协程）                      │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐     │
│  │        协程 1            │  │        协程 2           │     │
│  │    ticker_channel       │  │    trade_channel        │    │
│  │                         │  │                         │    │
│  │    await recv()         │  │    await recv()         │    │
│  │          ↓              │  │          ↓              │    │
│  │    on_ticker()          │  │    on_trade()           │    │
│  │          ↓              │  │          ↓              │    │
│  │    put_nowait()         │  │    put_nowait()         │    │
│  └────────────┬────────────┘  └────────────┬────────────┘    │
│               │                            │                 │
└───────────────┼────────────────────────────┼─────────────────┘
                │                            │
                ▼                            ▼
         ┌─────────────┐              ┌─────────────┐
         │ticker_queue │              │ trade_queue │
         │ asyncio.Q   │              │ asyncio.Q   │
         └──────┬──────┘              └──────┬──────┘
                │                            │
                ▼                            ▼
┌───────────────────────────────────────────────────────────────┐
│                   ServiceBase（消费者协程）                     │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐     │
│  │        协程 A            │  │        协程 B           │     │
│  │   process_tickers()     │  │   process_trades()      │    │
│  │                         │  │                         │    │
│  │  await queue.get()      │  │  await queue.get()      │    │
│  │          ↓              │  │          ↓              │    │
│  │  on_ticker(ticker)      │  │  on_trade(trade)        │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

## Observer 接口规范


* 观察者需要实现相应的队列属性

```python
class MyObserver:
    def __init__(self):
        self.ticker_queue = asyncio.Queue()        # Channel.TICKER
        self.trade_queue = asyncio.Queue()         # Channel.TRADE
        self.my_order_queue = asyncio.Queue()      # Channel.MY_ORDER
        self.my_order_batch_queue = asyncio.Queue() # Channel.MY_ORDER_BATCH
```

## 子类如何订阅一个Channel

通过`run_channel`会启动一个协程，连接到某个交易所的某个Channel，**下面是子类中启动一个Channel的典型逻辑，提供三个回调函数:**

* `on_packet`：收到json消息的回调
* `on_connected`：连接成功的回调
* `on_disconnected`：连接断开的回调

```python
def on_ticker(self, json_msg: dict):
    ticker = json_msg  # parse
    for ob in self._observers:
        ob.ticker_queue.put_nowait(ticker)

async def ticker_channel(self):
    async def subscribe_wrapper():
        return 'your subscribe message'

    params = {
            "host": "your websocket host",   # WebSocket主机地址
            "channel": Channel.TICKER,       # 当前频道
            "subscribe": subscribe_wrapper,  # 订阅消息的生成函数
            "on_packet": self.on_ticker,     # 收到消息后的处理回调
            "on_connected": None,            # 连接成功后的回调(可扩展)
            "on_disconnected": None          # 连接断开后的回调(可扩展)
    }
    await self.run_channel(**params)
```

## 如何启动一个Stream

```python
from hftpy.exchange import get_exchange_stream

# 创建stream
params = {
    "exchange": Exchange.KRAKEN,
    "currency_pairs": [CurrencyPair.BTC_USD, CurrencyPair.ETH_USD],
    "subscribe_channels": [Channel.TICKER, Channel.TRADE],
    "api_key": "your api key",
    "api_secret": "your api secret",
    "api_auth_data": "your auth data",
    "api_host": "your api host",
    "reconnect_times": "your reconnect policy",
    "subscribe_args": "subscribe args"
}
stream = get_exchange_stream(**params)
stream.bind_observer(service)

# 启动stream - 启动协程
asyncio.create_task(stream.start())
```

## 使用示例

!!! note "注意"
    
    * 如果只是订阅公共的Channel，不需要传递`api_key`和`api_secret`
    * Websocket Stream通常和Service集成在一起使用，但这并不是必须得。Websocket Stream也创建后可以独立使用，只需要绑定的Observer符合接口规范即可

```python hl_lines="39"
import asyncio
from hftpy.common import Exchange, CurrencyPair, Channel
from hftpy.exchange import get_exchange_stream


class DataHandler:
    """数据处理观察者"""

    def __init__(self):
        self.ticker_queue = asyncio.Queue()
        self.trade_queue = asyncio.Queue()

    async def process_tickers(self):
        """处理 Ticker 数据"""
        while True:
            ticker = await self.ticker_queue.get()
            print(f"收到 Ticker: {ticker.symbol} {ticker.bidPrice} / {ticker.askPrice}")

    async def process_trades(self):
        """处理 Trade 数据"""
        while True:
            trade = await self.trade_queue.get()
            print(f"收到 Trade: {trade.symbol} {trade.price} x {trade.volume}")


async def main():
    # 创建 Stream
    stream = get_exchange_stream(
        exchange=Exchange.KRAKEN,
        currency_pairs=[CurrencyPair.BTC_USD, CurrencyPair.ETH_USD],
        subscribe_channels=[Channel.TICKER, Channel.TRADE],
        api_key="",        # 公共频道不需要认证
        api_secret="",
        subscribe_args={}
    )

    # 创建观察者并绑定
    handler = DataHandler()
    stream.bind_observer(handler)

    # 启动所有任务
    await asyncio.gather(
        stream.start(),              # 启动 WebSocket 订阅
        handler.process_tickers(),   # 消费 Ticker 数据
        handler.process_trades(),    # 消费 Trade 数据
    )

if __name__ == "__main__":
    asyncio.run(main())
```

## 和Service集成

* 需要订阅交易所的数据，只需要在Service配置中增加如下配置，消息会自动路由到对应的`on_<type>`回调中
    * Service本身就是一个Observer，实现了Subscriber Observer要求的队列属性
    * 在启动的时候，Service底层会自动将配置信息转换成订阅参数，进行不同Channel的订阅


!!! note "注意"
    
    * 如果订阅的是私有的Channel，需要提供`api_key`和`api_secret`来创建REST API，因为大部分交易所建立Websocket私有链接的时候，都需要REST API来获取认证Token
    * Websocket Stream通常和Service集成在一起使用，但这并不是必须得。Websocket Stream也创建后可以独立使用，只需要绑定的Observer符合接口规范即可
  
**配置信息：**

```json hl_lines="22-23"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            
            "apis": [
                {
                    "exchange": "binance",
                    "api_key": "your api key",
                    "api_secret": "your api secret"
                }
            ],
            "streams": [
                {
                    "exchange": "binance",
                    "currency_pairs": [
                        "btc/usdt",
                        "eth/usdt"
                    ],
                    "subscribe_channels": [
                        "my_trade",
                        "my_order"
                    ]
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
    def on_my_order(self, my_order: MyOrder) -> None:
        pass

    def on_my_trade(self, my_trade: MyTrade) -> None:
        pass
```