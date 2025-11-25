# 类型定义

* 为了效率，系统内部所有的类型定义统一使用枚举而不是字符串，这会涉及到跨语言转换和网络传输的问题，需要有一套规则保证跨语言和网络传输的一致和高效
* 跨语言：我们在C++会定义一套enum的数据类型，这套数据类型通过pybind导出给Python使用
* 网络传输：在protobuf中，定义一套同样的enum数据类型，用于网络传输，保证定义的顺序一致，在传输两端可以直接进行类型的强制转换

## Python Enum vs IntEnum

* 在我们的系统中，涉及到跨语言的部分，并不直接使用Python中的enum，而是使用C++中定义的enum，通过pybind导出给Python来使用，这里只是为了明确下Python enum的行为
* 在Python中，`Enum`和`IntEnum`是两种常用的枚举类型

### 特性对比

| 特性 | Enum | IntEnum |
|------|------|---------|
| name | ✅ | ✅ |
| value | ✅ | ✅ |
| Color(1) | ✅ | ✅ |
| Color["RED"] | ✅ | ✅ |
| int(xxx) | ❌ | ✅ |
| 与 int 比较 | ❌ | ✅ |
| < > 比较 | ❌ | ✅ |


### 代码示例

* 高亮的三行是`IntEnum`独有的行为，我们通过pybind导出的enum行为和`IntEnum`一致，可以方便的和int做相互转换

```python hl_lines="22-24"
from enum import Enum, IntEnum

# 定义普通 Enum
class ColorEnum(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3

# 定义 IntEnum
class ColorIntEnum(IntEnum):
    RED = 1
    GREEN = 2
    BLUE = 3

# 共同特性
print(ColorEnum.RED.name)        # "RED"
print(ColorEnum.RED.value)       # 1
print(ColorEnum(1))              # ColorEnum.RED
print(ColorEnum["RED"])          # ColorEnum.RED

# IntEnum 独有特性
print(int(ColorIntEnum.RED))     # 1 (Enum 不支持)
print(ColorIntEnum.RED == 1)     # True (Enum 返回 False)
print(ColorIntEnum.RED < 2)      # True (Enum 会报错)
```

## C++

=== "C++"

    ```cpp
    enum class Exchange : uint8_t
    {
        UNKNOWN = 0,
        BINANCE = 1,
        COINBASE,
        KRAKEN,
        BITSTAMP,
        OKX,
        IB,
        BYBIT,
        BINANCE_FUTURE,
        OKX_SWAP,
        KUCOIN,
        KRAKEN_FUTURE,
        BINANCE_DELIVERY,
        GATE_FUTURE,
        BYBIT_FUTURE,
        UPBIT,
        BITGET,
        BITGET_FUTURE,
        GATE,
        KUCOIN_FUTURE,
        RPC
    };
    ```

=== "Protobuf"

    ```protobuf
    enum Exchange {
      EXCHANGE_UNKNOWN = 0;
      BINANCE = 1;
      COINBASE = 2;
      KRAKEN = 3;
      BITSTAMP = 4;
      OKX = 5;
      IB = 6;
      BYBIT = 7;
      BINANCE_FUTURE = 8;
      OKX_SWAP = 9;
      KUCOIN = 10;
      KRAKEN_FUTURE = 11;
      BINANCE_DELIVERY = 12;
      GATE_FUTURE = 13;
      BYBIT_FUTURE = 14;
      UPBIT = 15;
      BITGET = 16;
      BITGET_FUTURE = 17;
      GATE = 18;
      KUCOIN_FUTURE = 19;
      RPC = 20;
    }
    ```

=== "C++ PyBind"

    * 在导出的时候，需要加`py::arithmetic()`才有Python中IntEnum类型的行为

    ```cpp hl_lines="4"
    namespace py = pybind11;
    PYBIND11_MODULE(hft, m)
    {
        py::enum_<hft::Exchange>(m, "Exchange", py::arithmetic(), "HFT Exchange")
            .value("UNKNOWN", hft::Exchange::UNKNOWN)
            .value("BINANCE", hft::Exchange::BINANCE)
            .value("COINBASE", hft::Exchange::COINBASE)
            .value("KRAKEN", hft::Exchange::KRAKEN)
            .value("BITSTAMP", hft::Exchange::BITSTAMP)
            .value("OKX", hft::Exchange::OKX)
            .value("IB", hft::Exchange::IB)
            .value("BYBIT", hft::Exchange::BYBIT)
            .value("BINANCE_FUTURE", hft::Exchange::BINANCE_FUTURE)
            .value("OKX_SWAP", hft::Exchange::OKX_SWAP)
            .value("KUCOIN", hft::Exchange::KUCOIN)
            .value("KRAKEN_FUTURE", hft::Exchange::KRAKEN_FUTURE)
            .value("BINANCE_DELIVERY", hft::Exchange::BINANCE_DELIVERY)
            .value("GATE_FUTURE", hft::Exchange::GATE_FUTURE)
            .value("BYBIT_FUTURE", hft::Exchange::BYBIT_FUTURE)
            .value("UPBIT", hft::Exchange::UPBIT)
            .value("BITGET", hft::Exchange::BITGET)
            .value("BITGET_FUTURE", hft::Exchange::BITGET_FUTURE)
            .value("GATE", hft::Exchange::GATE)
            .value("KUCOIN_FUTURE", hft::Exchange::KUCOIN_FUTURE)
            .value("RPC", hft::Exchange::RPC)
            .export_values();
        m.def("exchangeToStr", &hft::exchangeToStr, py::arg("e"));
        m.def("strToExchange", &hft::strToExchange, py::arg("str"));
    }
    ```

### C++ -> Python

C++通过pybind导出的类型在Python中直接使用

```python hl_lines="4"
import hft
Exchange = hft.Exchange

exchange = Exchange.BINANCE
```

### C++ -> Protobuf

C++的类型转换成Protobuf的类型

=== "C++"

    ```c++
    struct alignas(CACHE_LINE_SIZE) Ticker
    {
        Exchange exchange;      // 交易所
        CurrencyPair symbol;    // 交易对
        int64_t timestamp;      // 服务器时间戳(微秒)
        int64_t localTimestamp; // 本地时间戳(微秒)
        double askPrice;        // 卖一价
        double askVolume;       // 卖一量
        double bidPrice;        // 买一价
        double bidVolume;       // 买一量
    };
    ```

=== "Protobuf"

    ```protobuf
    message Ticker {
      Exchange exchange = 1;     // 交易所
      CurrencyPair symbol = 2;   // 交易对
      int64 timestamp = 3;       // 服务器时间戳(微秒)
      int64 local_timestamp = 4; // 本地时间戳(微秒)
      double ask_price = 5;      // 卖一价
      double ask_volume = 6;     // 卖一量
      double bid_price = 7;      // 买一价
      double bid_volume = 8;     // 买一量
    }
    ```


* 由于enum定义的顺序一致，可以直接`static_cast`强制转换

```c++ hl_lines="3-4"
void tickerToProto(const Ticker &ticker, proto::Ticker &protoTicker)
{
    protoTicker.set_exchange(static_cast<proto::Exchange>(ticker.exchange));
    protoTicker.set_symbol(static_cast<proto::CurrencyPair>(ticker.symbol));
    protoTicker.set_timestamp(ticker.timestamp);
    protoTicker.set_local_timestamp(ticker.localTimestamp);
    protoTicker.set_ask_price(ticker.askPrice);
    protoTicker.set_ask_volume(ticker.askVolume);
    protoTicker.set_bid_price(ticker.bidPrice);
    protoTicker.set_bid_volume(ticker.bidVolume);
}
```

## Python

### Python -> C++

* Python和C++交互的唯一方式就是调用pybind导出的API，这些C++ API接口的参数会自动做类型转换

```python hl_lines="7"
import hft

Exchange = hft.Exchange
exchange_to_str = hft.exchangeToStr

exchange = Exchange.BINANCE
exchange_str = exchange_to_str(exchange)
```

### Python -> Protobuf

* Python的类型转换成Protobuf的类型，由于我们定义的顺序一致，可以直接强制转换，效率最高
* 由于pybind导出enum的时候增加了`py::arithmetic()`，导出的enum有类似IntEnum的行为，所以可以直接使用`int(exchange)`的方式做转换

```python hl_lines="3-4"
def hft_to_pb_ticker(hft_ticker: Ticker) -> pb.Ticker:
    pb_ticker = pb.Ticker()
    pb_ticker.exchange = int(hft_ticker.exchange)
    pb_ticker.symbol = int(hft_ticker.symbol)
    pb_ticker.timestamp = hft_ticker.timestamp
    pb_ticker.local_timestamp = hft_ticker.localTimestamp
    pb_ticker.ask_price = hft_ticker.askPrice
    pb_ticker.ask_volume = hft_ticker.askVolume
    pb_ticker.bid_price = hft_ticker.bidPrice
    pb_ticker.bid_volume = hft_ticker.bidVolume
    return pb_ticker
```

也可以用下面的方式做转换，效率比较低，不采用。优点是对两边enum定义的顺序没有要求，可以不一致

```python hl_lines="3-4"
def hft_to_pb_ticker(hft_ticker: Ticker) -> pb.Ticker:
    pb_ticker = pb.Ticker()
    pb_ticker.exchange = pb.Exchange.Value(hft_ticker.exchange.name)
    pb_ticker.symbol = pb.CurrencyPair.Value(hft_ticker.symbol.name)
    ...
```

## Protobuf

### Protobuf -> C++

* Protobuf的类型转换成C++的类型
* 由于enum定义的顺序一致，可以直接`static_cast`强制转换

```c++ hl_lines="4-5"
void protoToTicker(const proto::Ticker &protoTicker, Ticker &ticker)
{
    ticker.exchange = static_cast<Exchange>(protoTicker.exchange());
    ticker.symbol = static_cast<CurrencyPair>(protoTicker.symbol());
    ticker.timestamp = protoTicker.timestamp();
    ticker.localTimestamp = protoTicker.local_timestamp();
    ticker.askPrice = protoTicker.ask_price();
    ticker.askVolume = protoTicker.ask_volume();
    ticker.bidPrice = protoTicker.bid_price();
    ticker.bidVolume = protoTicker.bid_volume();
}
```

### Protobuf -> Python

* Protobuf的类型转换成Python的类型
* `Exchange(int)`的转换方式

```python hl_lines="4-5"
def pb_to_hft_ticker(pb_ticker: pb.Ticker) -> Ticker:
    return Ticker(
        exchange=Exchange(pb_ticker.exchange),
        symbol=CurrencyPair(pb_ticker.symbol),
        timestamp=pb_ticker.timestamp,
        localTimestamp=pb_ticker.local_timestamp,
        askPrice=pb_ticker.ask_price,
        askVolume=pb_ticker.ask_volume,
        bidPrice=pb_ticker.bid_price,
        bidVolume=pb_ticker.bid_volume
    )
```



