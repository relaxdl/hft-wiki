# 共享内存路径

## 三种类型的数据

* 不同的进程通过文件的inode信息生成跨进程共享的share memory key，**确保不同进程attach到同一块共享内存**

共享内存中有三大类数据，可以满足各种类型的策略需求

**类型一**

内部预定义的一些基础数据类型，例如：fair price，volatility，premium等

``` hl_lines="1"
/tmp/hft/shm/exchange.type.symbol.valueType
kraken.fair_price.usdt.pair_quote
```

**类型二**

市场数据类的，例如：trade，ticker，snapshot等

``` hl_lines="1"
/tmp/hft/shm/exchange.channel.symbol
kraken.trade.btcusd
```

**类型三**

任意自定义的中间信号，三级索引，每个索引的含义上层应用自己维护，例如：策略状态，需要跨进程共享的一些策略信号等

``` hl_lines="1"
/tmp/hft/shm/namespace.category.key.valueType
1.2.3.pair_quote
```


## 路径示例

下面是一些常见的路径

| 数据类型 | 标的 | 类型 | 路径示例 |
|---------|---------|---------|---------|
| 交易对的波动率 | CurrencyPair | 类型一 | `kraken.volatility.btcusd.pair_quote` |
| Currency的溢价 | Currency | 类型一 | `kraken.premium.btc.pair_quote` |
| Currency的资产 | Currency | 类型一 | `kraken.balance.usdt.single_quote` |
| Currency的Fair Price | Currency | 类型一 | `kraken.fair_price.usdt.pair_quote` |
| CurrencyPair的Fair Price | CurrencyPair | 类型一 | `kraken.fair_price.usdcusdt.pair_quote` |
| CurrencyPair的Trade | CurrencyPair | 类型二 | `kraken.trade.btcusd` |
| CurrencyPair的Ticker | CurrencyPair | 类型二 | `kraken.ticker.btcusd` |
| CurrencyPair的Snapshot | CurrencyPair | 类型二 | `kraken.snapshot.btcusd` |
| 自定义类型 | 自定义含义 | 类型三 | `1.2.3.pair_quote` |