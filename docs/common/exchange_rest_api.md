#REST API

## Python

* **工厂模式 + 单例模式**，KrakenAPI, BinanceAPI每个交易所的REST API全局只有一个实例。单例模式的好处是只需要在系统初始化的时候创建一次，后续在任何函数内可以直接调用。并且方便对进程内API的调用速率做全局控制
* **如何使用**：在系统初始化的时候，使用`create_exchange_api`创建一次api，之后就可以通过`get_exchange_api`获取使用了
* 调用REST API的私有接口，需要在创建的时候提供`api_key`和`api_secret`等认证信息。如果只调用共有接口，创建的时候不需要认证信息

### 类的继承结构

```
HttpBase (基类)
    ├── KrakenAPI
    ├── BinanceAPI
    ├── KrakenFutureAPI
    └── ...其他交易所API
```

### 创建和使用API

!!! note "注意"
    
    * 如果只是调用REST API的公共接口，不需要传递`api_key`和`api_secret`

```python
from hftpy.common import Exchange, CurrencyPair, Side
from hftpy.exchange import create_exchange_api, get_exchange_api

# 方式1: 系统初始化时创建
api = create_exchange_api(
    exchange=Exchange.KRAKEN,
    api_key="your_api_key",
    api_secret="your_api_secret"
)

# 方式2: 获取已创建的实例
api = get_exchange_api(Exchange.KRAKEN)

# 获取服务器时间
code, response = await api.get_time()

# 获取订单列表
code, orders = await api.get_open_orders()

# 下单示例
params = {
    "symbol": CurrencyPair.BTC_USDT,
    "side": Side.BUY,
    "amount": 0.01,
    "price": 87795.0,
    "client_order_id": short_uuid()
}
code, response = await rest_api.place_order(**params)
```

### 和Service的集成

* 只需要在service的配置文件中，增加需要的交易所配置信息，就可以直接使用。service在启动的时候，会根据配置，自动创建REST API。
* 下面的案例中，配置了2个交易所的认证信息，binance和coinbase的api在service中可以直接使用

**配置信息：**

```json hl_lines="7-16"
{
    "service.mm_demo": {
        "class_name": "MMDemoService",
        "setting": {
            ...
            "apis": [
                {
                    "exchange": "binance",
                    "api_key": "your_api_key",
                    "api_secret": "your_api_secret"
                },
                {
                    "exchange": "coinbase",
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

* service会自动创建`binance_api`和`coinbase_api`这两个成员变量，直接使用即可

```python
symbol = CurrencyPair.BTC_USDT
code, response = await self.binance_api.get_open_orders(symbol)
```

## C++

### 基类设计

  * 采用单例模式，每个交易所的rest api就是一个单例，允许实例化后直接使用
  * 提供基础的 HTTP 请求功能：`GET、POST、PUT、DELETE`
  * 发送请求时，仅设置公用的 HTTP 请求参数，个性化参数由子类自行配置
    * 可以通过实现`virtual function`的方式来改写父类的行为，把自己的个性化参数写入到curl的设置中
  * 包含回调接口，注册后可记录所有 HTTP 请求信息
    * 默认会写入到日志，子类可以自定义记录的行为

### 子类功能
  * 每个子类都有自己的`init`函数，用于设置`apiKey、apiSecret`等参数
  * 允许自定义发送 HTTP 请求的参数
    * 自己实现`virtual function`
  * 提供默认的请求头，支持自定义请求头
  * 实现认证模式，增加认证相关的请求头，适应交易所的认证需求

**这样的设计确保了基类的灵活性与可扩展性，同时让子类可以根据具体需求进行个性化定制**

### 使用方式

```c++
// 1. 初始化（系统启动时调用一次）
auto &api = hft::BinanceAPI::getInstance();
api.init("your_api_key", "your_api_secret", "", "");

// 2. 使用（任何地方直接调用）
sonic_json::Document result;
int code = hft::BinanceAPI::getInstance().getDepth(hft::CurrencyPair::BTC_USDT, 100, result);

if (code == 200) {
    // 成功，result 中包含深度数据
} else {
    // 失败处理
}
```