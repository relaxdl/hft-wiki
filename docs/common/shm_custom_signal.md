# 共享内存自定义信号

对于一些常用的基础数据类型，例如：Fair Price，Premium，Volatility，Balance等和一些常规的市场数据类型，例如：Trade，Ticker，Snapshot等。数据结构都是标准化的，**在共享内存中都做了预定义，可以直接使用**。我们还有另外一些业务场景，需要跨进程的共享一些自定义的数据类型，这时候预定义的数据类型不能满足业务需求，比如：一些策略计算的中间信号，多周期溢价的组合等也需要跨进程共享。这里我们设计了一套机制，每个策略可以根据自己的需求，自定义写入共享内存中数据的含义，上层策略自己能理解即可


## 设计对比

* 这里我们以`PairQuote`这个数据结构为例，对比下ShareMemoryPairQuote和ShareMemoryCustomPairQuote


| 维度 | ShareMemoryPairQuote | ShareMemoryCustomPairQuote |
|------|------------------------|------------------------------|
| 用途 | 内置数据类型 | 自定义因子/信号/策略数据（灵活扩展） |
| 参数类型 | 枚举类型（Exchange, DataType, Symbol） | int 类型（ns, category, key） |
| 重载版本 | 2个（CurrencyPair + Currency） | 1个 |
| 路径格式 | `exchange.dataType.symbol.valueType` | `ns.category.key.valueType` |

!!! note "注意"

    * 对于多数业务场景，ShareMemoryPairQuote通过内置的`DataType`大都可以覆盖到。需要额外扩展的业务场景，`（ns, category, key）`的含义业务逻辑自己去定义即可
    * 我们通过pybind导出的所有enum行为和IntEnum是一致的，可以直接转换为int，所以把Currency，CurrencyPair或者Exchange直接映射成ns，category，key都是可以的，根据业务需要，自行组合即可
    * 自定义信号最多支持三级索引，如果只需要一级或者2级所以，将ns或者category设置成0即可

## 案例

* 以PairQuote和Custom PairQuote为例，演示共享内存的读写操作

**示例：写入PairQuote**

=== "C++"

    ```cpp
    PairQuote quote;
    quote.timestamp = 1734500000000000;      // 16位微秒时间戳
    quote.localTimestamp = 1734500000000000;
    quote.ask = 50100.5;
    quote.bid = 50000.0;

    ShareMemoryPairQuote::write(
        Exchange::BINANCE,
        DataType::FAIR_PRICE,
        CurrencyPair::BTC_USDT,
        quote
    );
    ```

=== "Python"

    ```python
    import hft
               
    write_pair_quote_currency_pair = hft.writePairQuoteCurrencyPair

    quote = PairQuote(
        timestamp=1734500000000000,      # 16位微秒时间戳
        localTimestamp=1734500000000000,
        ask=50100.5,
        bid=50000.0
    )

    write_pair_quote_currency_pair(
        Exchange.BINANCE,
        DataType.FAIR_PRICE,
        CurrencyPair.BTC_USDT,
        quote
    )
    ```

**示例：读取PairQuote**

=== "C++"

    ```cpp
    PairQuote* quote = ShareMemoryPairQuote::get(
        Exchange::BINANCE,
        DataType::FAIR_PRICE,
        CurrencyPair::BTC_USDT
    );

    std::cout << "bid: " << quote->bid << ", ask: " << quote->ask << std::endl;
    ```

=== "Python"

    ```python
    import hft
                
    get_pair_quote_currency_pair = hft.getPairQuoteCurrencyPair  

    quote = get_pair_quote_currency_pair(
        Exchange.BINANCE,
        DataType.FAIR_PRICE,
        CurrencyPair.BTC_USDT
    )

    print(f"bid: {quote.bid}, ask: {quote.ask}")
    ```

**示例：写入Custom PairQuote**

=== "C++"

    ```cpp
    int ns = 1;        // 命名空间
    int category = 2;  // 类别
    int key = 101;     // 键名

    PairQuote quote;
    quote.timestamp = 1734500000000000;      // 16位微秒时间戳
    quote.localTimestamp = 1734500000000000;
    quote.ask = 50100.5;
    quote.bid = 50000.0;

    ShareMemoryCustomPairQuote::write(ns, category, key, quote);
    ```

=== "Python"

    ```python
    import hft

    write_custom_pair_quote = hft.writeCustomPairQuote

    ns = 1        # 命名空间
    category = 2  # 类别
    key = 101     # 键名

    quote = PairQuote(
        timestamp=1734500000000000,      # 16位微秒时间戳
        localTimestamp=1734500000000000,
        ask=50100.5,
        bid=50000.0
    )

    write_custom_pair_quote(ns, category, key, quote)
    ```

**示例：读取Custom PairQuote**

=== "C++"

    ```cpp
    int ns = 1;
    int category = 2;
    int key = 101;

    PairQuote* quote = ShareMemoryCustomPairQuote::get(ns, category, key);

    std::cout << "bid: " << quote->bid << ", ask: " << quote->ask << std::endl;
    ```

=== "Python"

    ```python
    import hft

    get_custom_pair_quote = hft.getCustomPairQuote

    ns = 1
    category = 2
    key = 101

    quote = get_custom_pair_quote(ns, category, key)

    print(f"bid: {quote.bid}, ask: {quote.ask}")
    ```