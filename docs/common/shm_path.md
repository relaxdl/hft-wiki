# 共享内存路径

* 不同的进程通过文件的inode信息生成跨进程共享的share memory key，确保不同进程attach到同一块共享内存

```
/tmp/hft/shm/exchange.type.symbol.valueType
```


## 路径示例

下面是一些常见的路径

| 数据类型 | 标的类型 | 路径示例 |
|---------|---------|---------|
| 交易对的波动率 | CurrencyPair | `kraken.volatility.btcusd.pair_quote` |
| Currency的溢价 | Currency | `kraken.premium.btc.pair_quote` |
| Currency的资产 | Currency | `kraken.balance.usdt.single_quote` |
| Currency的Fair Price | Currency | `kraken.fair_price.usdt.pair_quote` |
| CurrencyPair的Fair Price | CurrencyPair | `kraken.fair_price.usdcusdt.pair_quote` |
| CurrencyPair的Trade | CurrencyPair | `kraken.trade.btcusd` |
| CurrencyPair的Ticker | CurrencyPair | `kraken.ticker.btcusd` |
| CurrencyPair的Snapshot | CurrencyPair | `kraken.snapshot.btcusd` |