# 支持的平台

## 支持的交易所

* Market数据主要包括Public Channel的数据: Level1, Level2, Trade
  * Level2会实时合成Snapshot
* 个别的Public Channel数据需要认证才能获得高频推送, 例如: Coinbase的Level2

| 交易所名称 | 交易所ID | Market | 交易 |
| :--- | :--- | :---: | :---: |
| binance现货 | binance | ✅ | ✅ |
| binance future | binance_future | ✅ | ✅ |
| okx现货 | okx | ✅ | ✅ |
| okx future | okx_swap | ✅ | ✅ |
| coinbase现货 | coinbase | ✅ | ✅ |
| bitstamp现货 | bitstamp | ✅ | ✅ |
| kraken现货 | kraken | ✅ | ✅ |
| kraken future | kraken_future | ✅ | ✅ |
| bybit现货 | bybit | ✅ | 🚫 |
| bybit future | bybit_future | ✅ | 🚫 |
| kucoin现货 | kucoin | ✅ | 🚫 |
| kucoin future | kucoin_future | ✅ | 🚫 |
| bitget现货 | bitget | ✅ | ✅ |
| bitget future | bitget_future | ✅ | ✅ |

## 支持的操作系统

* 受不同供应商的限制, 需要支持不同的操作系统

| 操作系统 | 版本 | 是否支持 | 供应商 |
| :--- | :--- | :---: | :--- |
| CentOS | 8.5 | ✅ | aliyun |
| Amazon Linux | 2023 | ✅ | amazon aws (不支持CentOS) |
| Ubuntu Linux | 24.04 | ✅ | beeks (只支持Ubuntu) |
