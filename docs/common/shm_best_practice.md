# 共享内存最佳实践

## Fair Price

* Currency和CurrencyPair都支持fair price
* 底层C++的get/write方法提供了重载版本，可以传入 Currency（单币种）或 CurrencyPair（交易对），来读写其对应的fair price

```
kraken.fair_price.usdt.pair_quote
kraken.fair_price.usdcusdt.pair_quote
```

### Python中的接口形式

!!! note "注意"

    * 对于ShareMemoryPairQuote这个类，所有的方法全部是静态方法，通过pybind导出的时候，我们会导出一组全局函数，方便python一侧使用
    * 在C++中，get/write的时候使用重载版本没有问题，会根据参数是Currency还是CurrencyPair自动调用不同版本的函数。通过pybind把C++中定义的重载函数导出之后，**在python一侧无法通过参数是Currency还是CurrencyPair路由到正确的函数版本**
    * 我们在通过C++导出重载函数给python来使用的时候，最佳实践是对于每个不同的函数版本，给出不同的导出函数名，这样在python这一侧就不存在**重载**的概念，函数调用也不会有歧义
    * 在fair price的例子中，写入的时候，我们使用writePairQuoteCurrency来写入Currency的fair price，使用writePairQuoteCurrencyPair来写入CurrencyPair的fair price，不直接使用writePairQuote这个重载版本。读取的时候是同样的逻辑，使用getPairQuoteCurrency和getPairQuoteCurrencyPair来读取，而不直接使用getPairQuote这个重载版本

**下面是pybind的导出方式**

下面展示了部分导出的API，系统内部还有另外一套支持零拷贝的write接口


```c++ hl_lines="14 18 33 38"
namespace py = pybind11;
PYBIND11_MODULE(hft, m)
{
    // 绑定全局函数
    m.def("writePairQuote",
          static_cast<void (*)(const hft::Exchange, const hft::DataType, const hft::CurrencyPair, const hft::PairQuote &)>(
              &hft::ShareMemoryPairQuote::write),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"), py::arg("pairQuote"));
    m.def("writePairQuote",
          static_cast<void (*)(const hft::Exchange, const hft::DataType, const hft::Currency, const hft::PairQuote &)>(
              &hft::ShareMemoryPairQuote::write),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"), py::arg("pairQuote"));
    m.def("writePairQuoteCurrencyPair",
          static_cast<void (*)(const hft::Exchange, const hft::DataType, const hft::CurrencyPair, const hft::PairQuote &)>(
              &hft::ShareMemoryPairQuote::write),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"), py::arg("pairQuote"));
    m.def("writePairQuoteCurrency",
          static_cast<void (*)(const hft::Exchange, const hft::DataType, const hft::Currency, const hft::PairQuote &)>(
              &hft::ShareMemoryPairQuote::write),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"), py::arg("pairQuote"));

    m.def("getPairQuote",
          static_cast<hft::PairQuote (*)(const hft::Exchange, const hft::DataType, const hft::CurrencyPair)>(
              &hft::ShareMemoryPairQuote::getCopy),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"),
          py::return_value_policy::copy);
    m.def("getPairQuote",
          static_cast<hft::PairQuote (*)(const hft::Exchange, const hft::DataType, const hft::Currency)>(
              &hft::ShareMemoryPairQuote::getCopy),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"),
          py::return_value_policy::copy);
    m.def("getPairQuoteCurrencyPair",
          static_cast<hft::PairQuote (*)(const hft::Exchange, const hft::DataType, const hft::CurrencyPair)>(
              &hft::ShareMemoryPairQuote::getCopy),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"),
          py::return_value_policy::copy);
    m.def("getPairQuoteCurrency",
          static_cast<hft::PairQuote (*)(const hft::Exchange, const hft::DataType, const hft::Currency)>(
              &hft::ShareMemoryPairQuote::getCopy),
          py::arg("exchange"), py::arg("dataType"), py::arg("symbol"),
          py::return_value_policy::copy);
}
```

### Currency

**示例：写入Currency的Fair Price**

=== "C++"

    ```c++ hl_lines="7"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::FAIR_PRICE;
    Currency currency = Currency::BTC;
    PairQuote btcPrice = btcFairPrice->calculate(); // 计算btc的fair price
    ShareMemoryPairQuote::write(exchange, dataType, currency, btcPrice); // 更新共享内存
    ```

=== "Python"

    ```python hl_lines="20-28"
    import hft

    write_pair_quote_currency = hft.writePairQuoteCurrency                   
    write_pair_quote_currency_pair = hft.writePairQuoteCurrencyPair

    def write_fair_price(exchange: Exchange, symbol: Union[CurrencyPair, Currency], bid: float, ask: float, local_timestamp: int = None) -> None:
        if local_timestamp is None:
            local_timestamp = int(time.time() * 1_000_000)
        if isinstance(symbol, CurrencyPair):
            write_pair_quote_currency_pair(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
                PairQuote(timestamp=local_timestamp,
                          localTimestamp=local_timestamp,
                          ask=ask,
                          bid=bid)
            )
        elif isinstance(symbol, Currency):
            write_pair_quote_currency(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
                PairQuote(timestamp=local_timestamp,
                          localTimestamp=local_timestamp,
                          ask=ask,
                          bid=bid)
            )
    ```

**示例：获取Currency的Fair Price**

=== "C++"

    ```c++ hl_lines="6"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::FAIR_PRICE;
    Currency currency = Currency::BTC;
    PairQuote *btcPrice = ShareMemoryPairQuote::get(exchange, dataType, currency);
    
    // 输出bid和ask
    std::cout << "BTC Fair Price - Bid: " << btcPrice->bid 
              << ", Ask: " << btcPrice->ask << std::endl;
    ```

=== "Python"

    ```python hl_lines="13-17"
    import hft

    get_pair_quote_currency = hft.getPairQuoteCurrency                       
    get_pair_quote_currency_pair = hft.getPairQuoteCurrencyPair             
    def get_fair_price(exchange: Exchange, symbol: Union[CurrencyPair, Currency]) -> PairQuote:
        if isinstance(symbol, CurrencyPair):
            return get_pair_quote_currency_pair(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
            )
        elif isinstance(symbol, Currency):
            return get_pair_quote_currency(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
            )
    
    # 获取并输出BTC Fair Price
    btc_price = get_fair_price(Exchange.KRAKEN, Currency.BTC)
    print(f"BTC Fair Price - Bid: {btc_price.bid}, Ask: {btc_price.ask}")
    ```

### CurrencyPair

**示例：写入CurrencyPair的Fair Price**

=== "C++"

    ```c++ hl_lines="7"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::FAIR_PRICE;
    CurrencyPair symbol = Currency::USDT_USDC;
    PairQuote usdtUsdcPrice = usdtUsdcFairPrice->calculate(); // 计算usdtusdc的fair price
    ShareMemoryPairQuote::write(exchange, dataType, symbol, usdtUsdcPrice); // 更新共享内存
    ```

=== "Python"

    ```python hl_lines="10-18"
    import hft

    write_pair_quote_currency = hft.writePairQuoteCurrency                   
    write_pair_quote_currency_pair = hft.writePairQuoteCurrencyPair

    def write_fair_price(exchange: Exchange, symbol: Union[CurrencyPair, Currency], bid: float, ask: float, local_timestamp: int = None) -> None:
        if local_timestamp is None:
            local_timestamp = int(time.time() * 1_000_000)
        if isinstance(symbol, CurrencyPair):
            write_pair_quote_currency_pair(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
                PairQuote(timestamp=local_timestamp,
                          localTimestamp=local_timestamp,
                          ask=ask,
                          bid=bid)
            )
        elif isinstance(symbol, Currency):
            write_pair_quote_currency(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
                PairQuote(timestamp=local_timestamp,
                          localTimestamp=local_timestamp,
                          ask=ask,
                          bid=bid)
            )
    ```

**示例：获取CurrencyPair的Fair Price**

=== "C++"

    ```c++ hl_lines="6"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::FAIR_PRICE;
    CurrencyPair symbol = Currency::USDT_USDC;
    PairQuote *usdtUsdcPrice = ShareMemoryPairQuote::get(exchange, dataType, symbol);
    
    // 输出bid和ask
    std::cout << "USDT_USDC Fair Price - Bid: " << usdtUsdcPrice->bid 
              << ", Ask: " << usdtUsdcPrice->ask << std::endl;
    ```

=== "Python"

    ```python hl_lines="7-11"
    import hft

    get_pair_quote_currency = hft.getPairQuoteCurrency                       
    get_pair_quote_currency_pair = hft.getPairQuoteCurrencyPair             
    def get_fair_price(exchange: Exchange, symbol: Union[CurrencyPair, Currency]) -> PairQuote:
        if isinstance(symbol, CurrencyPair):
            return get_pair_quote_currency_pair(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
            )
        elif isinstance(symbol, Currency):
            return get_pair_quote_currency(
                exchange,
                DataType.FAIR_PRICE,
                symbol,
            )
    
    # 获取并输出USDT_USDC Fair Price
    usdt_usdc_price = get_fair_price(Exchange.KRAKEN, CurrencyPair.USDT_USDC)
    print(f"USDT_USDC Fair Price - Bid: {usdt_usdc_price.bid}, Ask: {usdt_usdc_price.ask}")
    ```

## Balance

```
kraken.balance.usdt.single_quote
```

**示例：写入Currency的Balance**

=== "C++"

    ```c++ hl_lines="8"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::BALANCE;
    Currency currency = Currency::USDT;
    int64_t now = getCurrentTimestamp();
    SingleQuote usdtBalance{now, now, 10000.0}; // 余额为10000 USDT
    ShareMemorySingleQuote::write(exchange, dataType, currency, usdtBalance); // 更新共享内存
    ```

=== "Python"

    ```python hl_lines="13-17"
    import hft
    import time

    write_single_quote_currency = hft.writeSingleQuoteCurrency

    def write_balance(exchange: Exchange, currency: Currency, balance: float, local_timestamp: int = None) -> None:
        if local_timestamp is None:
            local_timestamp = int(time.time() * 1_000_000)
        write_single_quote_currency(
            exchange,
            DataType.BALANCE,
            currency,
            SingleQuote(timestamp=local_timestamp,
                        localTimestamp=local_timestamp,
                        mid=balance)
        )
    
    # 写入USDT余额
    write_balance(Exchange.KRAKEN, Currency.USDT, 10000.0)
    ```

**示例：获取Currency的Balance**

=== "C++"

    ```c++ hl_lines="6"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::BALANCE;
    Currency currency = Currency::USDT;
    SingleQuote *usdtBalance = ShareMemorySingleQuote::get(exchange, dataType, currency);
    
    // 输出余额
    std::cout << "USDT Balance: " << usdtBalance->mid << std::endl;
    ```

=== "Python"

    ```python hl_lines="6-10"
    import hft

    get_single_quote_currency = hft.getSingleQuoteCurrency
    
    def get_balance(exchange: Exchange, currency: Currency) -> SingleQuote:
        return get_single_quote_currency(
            exchange,
            DataType.BALANCE,
            currency,
        )
    
    # 获取并输出USDT余额
    usdt_balance = get_balance(Exchange.KRAKEN, Currency.USDT)
    print(f"USDT Balance: {usdt_balance.mid}")
    ```


## Premium

```
kraken.premium.btc.pair_quote
```

**示例：写入Currency的Premium**

=== "C++"

    ```c++ hl_lines="7"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::PREMIUM;
    Currency currency = Currency::BTC;
    PairQuote btcPremium = btcPremiumCalculator->calculate(); // 计算btc的premium
    ShareMemoryPairQuote::write(exchange, dataType, currency, btcPremium); // 更新共享内存
    ```

=== "Python"

    ```python hl_lines="20-28"
    import hft

    write_pair_quote_currency = hft.writePairQuoteCurrency                   
    write_pair_quote_currency_pair = hft.writePairQuoteCurrencyPair

    def write_premium(exchange: Exchange, symbol: Union[CurrencyPair, Currency], bid: float, ask: float, local_timestamp: int = None) -> None:
        if local_timestamp is None:
            local_timestamp = int(time.time() * 1_000_000)
        if isinstance(symbol, CurrencyPair):
            write_pair_quote_currency_pair(
                exchange,
                DataType.PREMIUM,
                symbol,
                PairQuote(timestamp=local_timestamp,
                          localTimestamp=local_timestamp,
                          ask=ask,
                          bid=bid)
            )
        elif isinstance(symbol, Currency):
            write_pair_quote_currency(
                exchange,
                DataType.PREMIUM,
                symbol,
                PairQuote(timestamp=local_timestamp,
                          localTimestamp=local_timestamp,
                          ask=ask,
                          bid=bid)
            )
    ```

**示例：获取Currency的Premium**

=== "C++"

    ```c++ hl_lines="6"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::PREMIUM;
    Currency currency = Currency::BTC;
    PairQuote *btcPremium = ShareMemoryPairQuote::get(exchange, dataType, currency);
    
    // 输出bid和ask
    std::cout << "BTC Premium - Bid: " << btcPremium->bid 
              << ", Ask: " << btcPremium->ask << std::endl;
    ```

=== "Python"

    ```python hl_lines="13-17"
    import hft

    get_pair_quote_currency = hft.getPairQuoteCurrency                       
    get_pair_quote_currency_pair = hft.getPairQuoteCurrencyPair             
    def get_premium(exchange: Exchange, symbol: Union[CurrencyPair, Currency]) -> PairQuote:
        if isinstance(symbol, CurrencyPair):
            return get_pair_quote_currency_pair(
                exchange,
                DataType.PREMIUM,
                symbol,
            )
        elif isinstance(symbol, Currency):
            return get_pair_quote_currency(
                exchange,
                DataType.PREMIUM,
                symbol,
            )
    
    # 获取并输出BTC Premium
    btc_premium = get_premium(Exchange.KRAKEN, Currency.BTC)
    print(f"BTC Premium - Bid: {btc_premium.bid}, Ask: {btc_premium.ask}")
    ```


## Volatility

```
kraken.volatility.btcusd.pair_quote
```

**示例：写入CurrencyPair的Volatility**

=== "C++"

    ```c++ hl_lines="7"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::VOLATILITY;
    CurrencyPair symbol = CurrencyPair::BTC_USD;
    PairQuote volatility = volatilityCalculator->calculate(); // 计算波动率
    ShareMemoryPairQuote::write(exchange, dataType, symbol, volatility); // 更新共享内存
    ```

=== "Python"

    ```python hl_lines="9-17"
    import hft

    write_pair_quote_currency_pair = hft.writePairQuoteCurrencyPair

    def write_volatility(exchange: Exchange, symbol: CurrencyPair, bid: float, ask: float, local_timestamp: int = None) -> None:
        if local_timestamp is None:
            local_timestamp = int(time.time() * 1_000_000)
        
        write_pair_quote_currency_pair(
            exchange,
            DataType.VOLATILITY,
            symbol,
            PairQuote(timestamp=local_timestamp,
                      localTimestamp=local_timestamp,
                      ask=ask,
                      bid=bid)
        )
    ```

**示例：获取CurrencyPair的Volatility**

=== "C++"

    ```c++ hl_lines="6"
    #include "hft/util/shm.h"

    Exchange exchange = Exchange::KRAKEN;
    DataType dataType = DataType::VOLATILITY;
    CurrencyPair symbol = CurrencyPair::BTC_USD;
    PairQuote *volatility = ShareMemoryPairQuote::get(exchange, dataType, symbol);
    
    // 输出bid和ask
    std::cout << "BTC_USD Volatility - Bid: " << volatility->bid 
              << ", Ask: " << volatility->ask << std::endl;
    ```

=== "Python"

    ```python hl_lines="6-10"
    import hft

    get_pair_quote_currency_pair = hft.getPairQuoteCurrencyPair
    
    def get_volatility(exchange: Exchange, symbol: CurrencyPair) -> PairQuote:
        return get_pair_quote_currency_pair(
            exchange,
            DataType.VOLATILITY,
            symbol,
        )
    
    # 获取并输出BTC_USD Volatility
    volatility = get_volatility(Exchange.KRAKEN, CurrencyPair.BTC_USD)
    print(f"BTC_USD Volatility - Bid: {volatility.bid}, Ask: {volatility.ask}")
    ```

## Ticker

```
kraken.ticker.btcusd
```

**示例：写入CurrencyPair的Ticker**

=== "C++"

    ```c++ hl_lines="15"
    #include "hft/util/shm.h"

    // 构造 Ticker 对象
    Ticker ticker;
    ticker.exchange = Exchange::KRAKEN;
    ticker.symbol = CurrencyPair::BTC_USD;
    ticker.timestamp = getCurrentTimestamp();
    ticker.localTimestamp = getCurrentTimestamp();
    ticker.askPrice = 50000.0;
    ticker.askVolume = 1.5;
    ticker.bidPrice = 49999.0;
    ticker.bidVolume = 2.0;
    
    // 写入共享内存
    ShareMemoryTicker::write(ticker);
    ```

=== "Python"

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

```
kraken.trade.btcusd
```

**示例：写入CurrencyPair的Trade**

=== "C++"

    ```c++ hl_lines="16"
    #include "hft/util/shm.h"

    // 构造 Trade 对象
    Trade trade;
    trade.exchange = Exchange::KRAKEN;
    trade.symbol = CurrencyPair::BTC_USD;
    trade.timestamp = getCurrentTimestamp();
    trade.localTimestamp = getCurrentTimestamp();
    trade.tradeTimestamp = getCurrentTimestamp();
    trade.side = Side::BUY;
    trade.price = 50000.0;
    trade.volume = 0.5;
    
    // 写入共享内存
    ShareMemoryTrade::write(trade);
    ```

=== "Python"

    ```python hl_lines="16"
    import hft
    
    # 构造 Trade 对象
    trade = hft.Trade()
    trade.exchange = hft.Exchange.KRAKEN
    trade.symbol = hft.CurrencyPair.BTC_USD
    trade.timestamp = int(time.time() * 1_000_000)
    trade.localTimestamp = trade.timestamp
    trade.tradeTimestamp = trade.timestamp
    trade.side = hft.Side.BUY
    trade.price = 50000.0
    trade.volume = 0.5
    
    # 写入共享内存
    hft.writeTrade(trade)
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

```
kraken.snapshot.btcusd
```

**示例：写入CurrencyPair的Snapshot**

=== "C++"

    ```c++ hl_lines="27"
    #include "hft/util/shm.h"

    // 构造 Snapshot 对象
    Snapshot snapshot;
    snapshot.exchange = Exchange::KRAKEN;
    snapshot.symbol = CurrencyPair::BTC_USD;
    snapshot.timestamp = getCurrentTimestamp();
    snapshot.localTimestamp = getCurrentTimestamp();
    
    // 卖盘（asks）- 价格从低到高
    snapshot.askCount = 5;
    snapshot.asksPrice[0] = 50001.0;
    snapshot.asksVolume[0] = 1.0;
    snapshot.asksPrice[1] = 50002.0;
    snapshot.asksVolume[1] = 2.0;
    // ... 更多卖盘档位
    
    // 买盘（bids）- 价格从高到低
    snapshot.bidCount = 5;
    snapshot.bidsPrice[0] = 50000.0;
    snapshot.bidsVolume[0] = 1.5;
    snapshot.bidsPrice[1] = 49999.0;
    snapshot.bidsVolume[1] = 2.5;
    // ... 更多买盘档位
    
    // 写入共享内存
    ShareMemorySnapshot::write(snapshot);
    ```

=== "Python"

    ```python
    import hft
    import time
    from hftpy.common import PySnapshot
    
    # 构造 PySnapshot 对象（Python 自定义的快照结构）
    # Python一侧无法写入共享内存
    py_snapshot = PySnapshot(
        exchange=hft.Exchange.KRAKEN,
        symbol=hft.CurrencyPair.BTC_USD,
        timestamp=int(time.time() * 1_000_000),
        localTimestamp=int(time.time() * 1_000_000),
        askCount=5,
        bidCount=5,
        asksPrice=[50001.0, 50002.0, 50003.0, 50004.0, 50005.0],
        asksVolume=[1.0, 2.0, 1.5, 3.0, 2.5],
        bidsPrice=[50000.0, 49999.0, 49998.0, 49997.0, 49996.0],
        bidsVolume=[1.5, 2.5, 1.8, 3.5, 2.8]
    )
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