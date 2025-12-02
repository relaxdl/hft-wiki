# 数据结构

* 跨语言：所有的数据结构都在C++中定义，通过pybind导出给Python使用
* 网络传输：在protobuf中，定义一套同样的数据结构，用于网络传输，可以通过自定义的类型转换方法与C++和Python中的结构进行相互转换

## C++

* 下面以`Ticker`为例说明数据结构的定义方式

!!! note "注意"
    
    * C++中的成员变量使用`lowerCamelCase`命名规则，所以导出给Python使用的时候，也是`lowerCamelCase`的命名风格
    * Protobuf中的成员变量使用官方推荐的`snake_case`命名风格

=== "C++"

    ```cpp
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

=== "C++ PyBind"

    ```cpp
    namespace py = pybind11;
    PYBIND11_MODULE(hft, m)
    {
        py::class_<hft::Ticker>(m, "Ticker")
            .def(py::init([](hft::Exchange exchange, hft::CurrencyPair symbol, int64_t timestamp, int64_t localTimestamp,
                             double askPrice, double askVolume, double bidPrice, double bidVolume)
                          { return hft::Ticker{exchange, symbol, timestamp, localTimestamp, askPrice, askVolume, bidPrice, bidVolume}; }),
                 py::arg("exchange") = hft::Exchange::UNKNOWN,
                 py::arg("symbol") = hft::CurrencyPair::UNKNOWN,
                 py::arg("timestamp") = 0,
                 py::arg("localTimestamp") = 0,
                 py::arg("askPrice") = 0.0,
                 py::arg("askVolume") = 0.0,
                 py::arg("bidPrice") = 0.0,
                 py::arg("bidVolume") = 0.0)
            .def_readwrite("exchange", &hft::Ticker::exchange)
            .def_readwrite("symbol", &hft::Ticker::symbol)
            .def_readwrite("timestamp", &hft::Ticker::timestamp)
            .def_readwrite("localTimestamp", &hft::Ticker::localTimestamp)
            .def_readwrite("askPrice", &hft::Ticker::askPrice)
            .def_readwrite("askVolume", &hft::Ticker::askVolume)
            .def_readwrite("bidPrice", &hft::Ticker::bidPrice)
            .def_readwrite("bidVolume", &hft::Ticker::bidVolume)
            .def("__repr__", [](const hft::Ticker &t)
                 { return tickerToStr(t); });
        m.def("tickerToStr", &hft::tickerToStr, "ticker to str");
    }
    ```

### C++ -> Python

C++通过pybind导出的数据结构，在Python中直接使用

```python
import hft
Exchange = hft.Exchange
CurrencyPair = hft.CurrencyPair
Ticker = hft.Ticker

ticker = Ticker(
    exchange=Exchange.BINANCE,
    symbol=CurrencyPair.BTC_USDT,
    timestamp=int(time.time() * 1e6),
    localTimestamp=int(time.time() * 1e6),
    askPrice=110101.2,
    askVolume=1.0,
    bidPrice=110101.1,
    bidVolume=2.0
)
```

### C++ -> Protobuf

* C++的数据结构转换成Python的数据结构

```c++
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

* Python的数据结构转换成C++的数据结构
* Python和C++交互的唯一方式就是调用pybind导出的API，这些C++ API接口的参数会自动做数据类型的转换

```python hl_lines="15"
import hft

# 构造 Ticker 对象
ticker = hft.Ticker()
ticker.exchange = hft.Exchange.KRAKEN
ticker.symbol = hft.CurrencyPair.BTC_USD
ticker.timestamp = int(time.time() * 1_000_000)
ticker.localTimestamp = ticker.timestamp
ticker.askPrice = 50000.0
ticker.askVolume = 1.5
ticker.bidPrice = 49999.0
ticker.bidVolume = 2.0

# 写入共享内存
hft.writeTicker(ticker)
```

### Python -> Protobuf

* Python的数据结构转换成Protobuf的数据结构

```python
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

## Protobuf

### Protobuf -> C++

* Protobuf的数据结构转换成C++的数据结构

```c++
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

* Protobuf的数据结构转换成Python的数据结构

```python
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
