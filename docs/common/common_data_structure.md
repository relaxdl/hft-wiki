# 常用数据结构

* 这些数据结构都在C++中定义，通过pybind导出给Python使用
* C++提供了API，可以直接读取共享内存的这些数据结构


## Ticker

* 一档的买卖价格和数量

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

=== "C++ PyBind"

    ```c++
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

**示例：获取CurrencyPair的Ticker**

=== "C++"

    ```c++ hl_lines="5"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    CurrencyPair symbol = CurrencyPair::BTC_USD;
    Ticker *ticker = ShareMemoryTicker::get(exchange, symbol);
    
    // 输出行情数据
    std::cout << "BTC_USD Ticker - Ask: " << ticker->askPrice 
              << " (" << ticker->askVolume << "), "
              << "Bid: " << ticker->bidPrice 
              << " (" << ticker->bidVolume << ")" << std::endl;
    ```

=== "Python"

    ```python hl_lines="5"
    import hft
    
    exchange = hft.Exchange.KRAKEN
    symbol = hft.CurrencyPair.BTC_USD
    ticker = hft.getTicker(exchange, symbol)
    
    # 输出行情数据
    print(f"BTC_USD Ticker - Ask: {ticker.askPrice} ({ticker.askVolume}), "
          f"Bid: {ticker.bidPrice} ({ticker.bidVolume})")
    ```


## Trade

* 市场的Trade数据

=== "C++"

    ```c++
    struct alignas(CACHE_LINE_SIZE) Trade
    {
        Exchange exchange;      // 交易所
        CurrencyPair symbol;    // 交易对
        int64_t timestamp;      // 服务器时间戳(微秒)
        int64_t localTimestamp; // 本地时间戳(微秒)
        int64_t tradeTimestamp; // 成交时间戳(微秒)
        Side side;              // 买卖方向
        double price;           // 成交价格
        double volume;          // 成交量
    };
    ```

=== "C++ PyBind"

    ```c++
    namespace py = pybind11;
    PYBIND11_MODULE(hft, m)
    {
        py::class_<hft::Trade>(m, "Trade")
            .def(py::init([](hft::Exchange exchange, hft::CurrencyPair symbol, int64_t timestamp, int64_t localTimestamp,
                                int64_t tradeTimestamp, hft::Side side, double price, double volume)
                            { return hft::Trade{exchange, symbol, timestamp, localTimestamp, tradeTimestamp, side, price, volume}; }),
                    py::arg("exchange") = hft::Exchange::UNKNOWN,
                    py::arg("symbol") = hft::CurrencyPair::UNKNOWN,
                    py::arg("timestamp") = 0,
                    py::arg("localTimestamp") = 0,
                    py::arg("tradeTimestamp") = 0,
                    py::arg("side") = hft::Side::UNKNOWN,
                    py::arg("price") = 0.0,
                    py::arg("volume") = 0.0)
            .def_readwrite("exchange", &hft::Trade::exchange)
            .def_readwrite("symbol", &hft::Trade::symbol)
            .def_readwrite("timestamp", &hft::Trade::timestamp)
            .def_readwrite("localTimestamp", &hft::Trade::localTimestamp)
            .def_readwrite("tradeTimestamp", &hft::Trade::tradeTimestamp)
            .def_readwrite("side", &hft::Trade::side)
            .def_readwrite("price", &hft::Trade::price)
            .def_readwrite("volume", &hft::Trade::volume)
            .def("__repr__", [](const hft::Trade &t)
                    { return tradeToStr(t); });
        m.def("tradeToStr", &hft::tradeToStr, "trade to str");
    }
    ```


**示例：获取CurrencyPair的Trade**

=== "C++"

    ```c++ hl_lines="5"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    CurrencyPair symbol = CurrencyPair::BTC_USD;
    Trade *trade = ShareMemoryTrade::get(exchange, symbol);
    
    // 输出成交数据
    std::cout << "BTC_USD Trade - Side: " << sideToStr(trade->side)
              << ", Price: " << trade->price
              << ", Volume: " << trade->volume
              << ", Time: " << trade->tradeTimestamp << std::endl;
    ```

=== "Python"

    ```python hl_lines="5"
    import hft
    
    exchange = hft.Exchange.KRAKEN
    symbol = hft.CurrencyPair.BTC_USD
    trade = hft.getTrade(exchange, symbol)
    
    # 输出成交数据
    print(f"BTC_USD Trade - Side: {trade.side}, "
          f"Price: {trade.price}, Volume: {trade.volume}, "
          f"Time: {trade.tradeTimestamp}")
    ```

## Snapshot

* 订单册快照，每个深度的价格和数量
* 交易系统底层会把数据流实时合成订单册快照，写入共享内存，上层应用可以直接读取最新的订单册快照
* 底层C++在填充Snapshot的时候, askCount、bidCount范围内的数据保证有效，超出askCount、bidCount的部分不保证数据有效性，使用时需自行根据count判断数据范围

=== "C++"

    ```c++
    struct alignas(CACHE_LINE_SIZE) Snapshot
    {
        Exchange exchange;      // 交易所
        CurrencyPair symbol;    // 交易对
        int64_t timestamp;      // 服务器时间戳(微秒)
        int64_t localTimestamp; // 本地时间戳(微秒)
        int askCount;           // 卖档数量
        int bidCount;           // 买档数量

        static constexpr int MAX_DEPTH = 256;
        double asksPrice[MAX_DEPTH];  // 卖价列表
        double asksVolume[MAX_DEPTH]; // 卖量列表
        double bidsPrice[MAX_DEPTH];  // 买价列表
        double bidsVolume[MAX_DEPTH]; // 买量列表
    };
    ```

=== "C++ PyBind"

    ```c++
    namespace py = pybind11;
    PYBIND11_MODULE(hft, m)
    {
        py::class_<hft::Snapshot>(m, "Snapshot")
        .def(py::init<>())
        .def_readonly("exchange", &hft::Snapshot::exchange)
        .def_readonly("symbol", &hft::Snapshot::symbol)
        .def_readonly("timestamp", &hft::Snapshot::timestamp)
        .def_readonly("localTimestamp", &hft::Snapshot::localTimestamp)
        .def_readonly("askCount", &hft::Snapshot::askCount)
        .def_readonly("bidCount", &hft::Snapshot::bidCount)
        .def_property_readonly("asksPrice",
                                [](const hft::Snapshot &s)
                                {
                                    return py::array_t<double>(
                                        {hft::Snapshot::MAX_DEPTH}, {sizeof(double)}, s.asksPrice, py::capsule([]() {}));
                                })
        .def_property_readonly("asksVolume",
                                [](const hft::Snapshot &s)
                                {
                                    return py::array_t<double>(
                                        {hft::Snapshot::MAX_DEPTH}, {sizeof(double)}, s.asksVolume, py::capsule([]() {}));
                                })
        .def_property_readonly("bidsPrice",
                                [](const hft::Snapshot &s)
                                {
                                    return py::array_t<double>(
                                        {hft::Snapshot::MAX_DEPTH}, {sizeof(double)}, s.bidsPrice, py::capsule([]() {}));
                                })
        .def_property_readonly("bidsVolume",
                                [](const hft::Snapshot &s)
                                {
                                    return py::array_t<double>(
                                        {hft::Snapshot::MAX_DEPTH}, {sizeof(double)}, s.bidsVolume, py::capsule([]() {}));
                                })
        .def("__repr__", [](const hft::Snapshot &s)
                { return snapshot5ToStr(s); });
        m.def("snapshot5ToStr", &hft::snapshot5ToStr, "snapshot5 to str");
    }
    ```


**示例：获取CurrencyPair的Snapshot**

=== "C++"

    ```c++ hl_lines="5"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    CurrencyPair symbol = CurrencyPair::BTC_USD;
    Snapshot *snapshot = ShareMemorySnapshot::get(exchange, symbol);
    
    // 输出快照数据
    std::cout << "BTC_USD Snapshot:" << std::endl;
    std::cout << "Ask Count: " << snapshot->askCount << std::endl;
    std::cout << "Best Ask: " << snapshot->asksPrice[0] 
              << " (" << snapshot->asksVolume[0] << ")" << std::endl;
    std::cout << "Bid Count: " << snapshot->bidCount << std::endl;
    std::cout << "Best Bid: " << snapshot->bidsPrice[0] 
              << " (" << snapshot->bidsVolume[0] << ")" << std::endl;
    ```

=== "Python"

    ```python hl_lines="5"
    import hft
    
    exchange = hft.Exchange.KRAKEN
    symbol = hft.CurrencyPair.BTC_USD
    snapshot = hft.getSnapshot(exchange, symbol)
    
    # 输出快照数据
    print(f"BTC_USD Snapshot:")
    print(f"Ask Count: {snapshot.askCount}")
    print(f"Best Ask: {snapshot.asksPrice[0]} ({snapshot.asksVolume[0]})")
    print(f"Bid Count: {snapshot.bidCount}")
    print(f"Best Bid: {snapshot.bidsPrice[0]} ({snapshot.bidsVolume[0]})")
    
    # 遍历所有卖盘档位
    print(f"\nAsks (Top 5):")
    for i in range(min(5, snapshot.askCount)):
        print(f"  Level {i+1}: {snapshot.asksPrice[i]} @ {snapshot.asksVolume[i]}")
    
    # 遍历所有买盘档位
    print(f"\nBids (Top 5):")
    for i in range(min(5, snapshot.bidCount)):
        print(f"  Level {i+1}: {snapshot.bidsPrice[i]} @ {snapshot.bidsVolume[i]}")
    ```

### PySnapshot

为什么要有`PySnapshot`这个结构？因为C++导出的`Snapshot`这个结构，**所有字段都是只读的**，不可以修改。这在实盘的场景中是没有问题的。我们实盘的场景是底层C++合成订单册快照，写入共享内存。上层的python应用只需要读取共享内存中的订单册快照进行使用，并不需要修改订单册快照。如果python一侧要做一些其他的逻辑，例如：用历史数据自己合成订单册，可以使用`PySnapshot`这个结构

```python
import hft
Snapshot = hft.Snapshot # C++定义的snapshot

# python中定义的snapshot
class PySnapshot(msgspec.Struct):
    exchange: Exchange          # 交易所
    symbol: CurrencyPair        # 交易对
    timestamp: int              # 服务器时间戳(微秒)
    localTimestamp: int        # 本地时间戳(微秒)
    askCount: int              # 卖档数量
    bidCount: int              # 买档数量
    asksPrice: List[float]     # 卖价列表
    asksVolume: List[float]    # 卖量列表
    bidsPrice: List[float]     # 买价列表
    bidsVolume: List[float]    # 买量列表
```

`PySnapshot`这个结构的字段和`Snapshot`完全一致，对于使用者来说，传入`PySnapshot`和`Snapshot`是没有任何区别的。

```python
def foo(snapshot: Union[PySnapshot, Snapshot]):
    pass
```