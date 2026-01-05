#Websocket API

## 设计特点

* **工厂模式 + 单例模式**，KrakenWsAPI，BinanceWsAPI Websocket API全局只有一个实例。单例模式的好处是只需要在系统初始化的时候创建一次，后续在任何函数内可以直接调用。并且方便对进程内API的调用速率做全局控制。同时能保证请求的顺序一致性
* **如何使用**：在系统初始化的时候，使用`create_exchange_ws_api`创建一次api，之后就可以通过`get_exchange_ws_api`获取使用了
* 由于大多数交易所的Websocket API需要REST API配置认证，所以通常在创建Websocket API的时候，先创建一个同所的REST API

## 创建和使用API

```python hl_lines="17-21 29-47"
import asyncio
from hftpy.common import CurrencyPair, Side, Exchange
from hftpy.exchange import create_exchange_api, create_exchange_ws_api, get_exchange_ws_api


def on_response(
    request_id: int,
    method: str,
    request: dict,
    response: dict,
    latency: int
):
    from hftpy.exchange.ws_api import WsApiBase
    WsApiBase.default_ws_callback(request_id, method, request, response, latency)


async def main():
    params = {
        "exchange": Exchange.KRAKEN,
        "api_key": "your_api_key",
        "api_secret": "your_api_secret"
    }
    # 创建 Rest API 实例
    create_exchange_api(**params)

    # 创建 WebSocket API 实例
    ws_api = create_exchange_ws_api(**params)

    # 启动 WebSocket 连接
    asyncio.create_task(ws_api.start())

    # 等待连接建立
    await asyncio.sleep(3.0)

    # 获取已创建的实例
    ws_api = get_exchange_ws_api(Exchange.KRAKEN)

    # 下单
    await ws_api.place_order(
        symbol=CurrencyPair.BTC_USD,
        side=Side.BUY,
        amount=0.001,
        price=80000.0,
        order_type="limit",
        post_only=True,
        callback=on_response
    )

    # 撤单
    await ws_api.cancel_order(
        order_id="your_order_id",
        callback=on_response
    )

    # 等待响应
    await asyncio.sleep(3.0)

if __name__ == "__main__":
    asyncio.run(main())
```

**▶ 输出：**

```
WebSocket Request - ID: 2 | Method: add_order | Latency: 12958μs
Request Params: {
    'token': 'nlFbymk2AVKtMpXTyq7eMDadYkTdyzeOK71oSfmKoDA',
    'order_type': 'limit',
    'post_only': True,
    'symbol': 'BTC/USD',
    'order_qty': 0.001,
    'side': 'buy',
    'limit_price': 80000.0,
    'time_in_force': 'gtc'
}
Response Data: {
    'method': 'add_order',
    'req_id': 2,
    'result': {'order_id': 'OK4GZK-XOTOY-U4C4X5'},
    'success': True,
    'time_in': '2026-01-05T07:48:36.492452Z',
    'time_out': '2026-01-05T07:48:36.500864Z'
}
```

```
WebSocket Request - ID: 3 | Method: cancel_order | Latency: 12642μs
Request Params: {
    'token': 'AHYuRN+G82kDMQBe8tqcxYPsUb2r86Hork2TqY3DJq0',
    'order_id': ['OK4GZK-XOTOY-U4C4X5']
}
Response Data: {
    'method': 'cancel_order',
    'req_id': 3,
    'result': {'order_id': 'OK4GZK-XOTOY-U4C4X5'},
    'success': True,
    'time_in': '2026-01-05T07:48:55.895607Z',
    'time_out': '2026-01-05T07:48:55.903683Z'
}
```

## 和Service的集成

* 只需要在service的配置文件中，增加需要的交易所配置信息，就可以直接使用。service在启动的时候，会根据配置，自动创建REST API & Websocket API
* 下面的案例中，会自动创建KrakenAPI和KrakenWsAPI

**配置信息：**

```json hl_lines="7-16"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "apis": [
                {
                    "exchange": "kraken",
                    "api_key": "your_api_key",
                    "api_secret": "your_api_secret"
                }
            ],
            "ws_apis": [
                {
                    "exchange": "kraken",
                    "api_key": "your_api_key",
                    "api_secret": "your_api_secret"
                }
            ],
            ...
        }
    }
}
```


**案例：**

* service会自动创建`kraken_api`和`kraken_ws_api`这两个成员变量，直接使用即可

```python
class MMDemoService(ServiceBase):
        symbol=CurrencyPair.BTC_USD,
        side=Side.BUY,
        amount=0.001,
        price=80000.0,
        order_type="limit",
        post_only=True,
        callback=on_response

    # ws下单回调函数
    def on_ws_place_order_callback(self, request_id, method, request, response, latency):
        """
        下maker单成功:
        {
            'token': '+zoUqE6bqQclouXTUmuchmCxMaRNqmdZYRdSwQe7u0g',
            'order_type': 'limit',
            'post_only': True,
            'symbol': 'EUR/USD',
            'order_qty': 1.0,
            'side': 'buy',
            'limit_price': 1.01,
            'cl_ord_id': '636a0574df7c4aa09c3b119a97f7c58b',
            'time_in_force': 'gtc'
        }
        {
            'method': 'add_order',
            'req_id': 3,
            'result': {
                'cl_ord_id': '636a0574df7c4aa09c3b119a97f7c58b',
                'order_id': 'OJZGTY-IZ3D4-EVVSTJ'
            },
            'success': True,
            'time_in': '2025-07-24T00:57:21.688509Z',
            'time_out': '2025-07-24T00:57:21.691084Z'
        }

        下maker单失败:
        {
            'token': '+zoUqE6bqQclouXTUmuchmCxMaRNqmdZYRdSwQe7u0g',
            'order_type': 'limit',
            'post_only': True,
            'symbol': 'EUR/USD',
            'order_qty': 0.1,
            'side': 'buy',
            'limit_price': 1.01,
            'cl_ord_id': 'f1c24e1f0fa44726add4999b2e086b3d',
            'time_in_force': 'gtc'
        }
        {
            'error': 'EGeneral:Invalid arguments:volume minimum not met',
            'method': 'add_order',
            'req_id': 2,
            'success': False,
            'time_in': '2025-07-24T00:56:33.320028Z',
            'time_out': '2025-07-24T00:56:33.328907Z'
        }
        """
        if isinstance(response, dict) and response.get("success"):
            # 下单成功，可以进行后续处理
            pass
        else:
            # 下单失败, 把订单从本地删除
            if isinstance(response, dict) and response.get("success") is False:
                client_order_id = request.get("cl_ord_id")
                # 获取老的订单信息
                my_order = self._orders.get(client_order_id, None)
                self._orders.pop(client_order_id, None)
            logger.warning(f"ws_place_order:{request_id}:{method}:{request}:{response}:{latency}")

    async def ws_place_order(self):
        params = {
            "symbol": CurrencyPair.BTC_USD,
            "side": Side.BUY,
            "amount": 0.001,
            "price": 80000.0,
            "order_type": "limit",
            "post_only": True,
            "req_id": self.kraken_ws_api.get_next_request_id(),
            "callback": self.on_ws_place_order_callback
        }
        await self.kraken_ws_api.place_order(**params)
```
