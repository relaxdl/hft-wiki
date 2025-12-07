# 转账

## 账户体系

转账系统基于底层的账户体系来工作

### 账户命名规范

* 采用三段式命名：交易所.账户名.钱包类型
* **账户名**是系统内部为每个账户自定义的名字，在转账的时候，用这个名字可以直接转账。同一个交易所的同一个账户名，对应一个子账户

!!! note "注意"

    * 这是一个简化的抽象，交易所实际的账户体系比这个更复杂，这个简化的抽象可以让转账系统更方便的工作
    * 三段式的命名，对应到API的接口上，用2个字段就可以描述。交易所和钱包类型，可以用`Exchange`来标识，`Exchange.KRAKEN, Exchange.KRAKEN_FUTURE`，账户名称可以是任意预先定义好的字符串
    * **同一个交易所，不同子账户之间转账，在底层需要有唯一的标识来标识不同的子账户**。不同的交易所需要的配置不一样，例如：binance每个子账户需要配置一个email信息，kraken每个子账户需要配置一个public account id，kraken_future每个子账户需要配置一个uid等。这些信息在底层配置好之后，上层应用在使用API的时候不需要关心转账系统底层的这些差异

```
kraken.trade.spot
   │     │     │
   │     │     └── 钱包类型: spot / future
   │     └──────── 账户名称: main / trade / future / close_position
   └────────────── 交易所: kraken / binance
```

### 账户定义案例

**账户名称**

```python
# Kraken 账户名
KrakenAccountType = Literal["main", "trade", "future", "close_position"]

# Binance 账户名
BinanceAccountType = Literal["main", "trade"]

# 统一账户名
AccountType = Union[KrakenAccountType, BinanceAccountType]
```

**Exchange 枚举与钱包类型**

```python
Exchange.BINANCE       # → Binance Spot
Exchange.KRAKEN        # → Kraken Spot
Exchange.KRAKEN_FUTURE # → Kraken Future
```

**通过 Exchange + AccountType 的组合，可以唯一确定一个账户：**

```python
(Exchange.KRAKEN, "trade")        # → kraken.trade.spot
(Exchange.KRAKEN_FUTURE, "main")  # → kraken.main.future
(Exchange.BINANCE, "trade")       # → binance.trade.spot
```

### 账户配置案例

下面是具体的案例

#### Binance 账户

| 账户标识 | 账户名 | 用途 |
|----------|----------|------|
| binance.main.spot | main | 跨交易所提币出入口 |
| binance.trade.spot | trade | 稳定币套利交易 |

binance账户的底层配置信息

| 名称 | 用途 |
|----------|----------|
| api_key + api_secret | API 请求签名认证 |
| email | Spot 账户间转账时标识目标账户 |

#### Kraken 账户

| 账户标识 | 账户名 | 用途 |
|----------|----------|------|
| kraken.main.spot | main | 跨交易所提币出入口 |
| kraken.trade.spot | trade | 日常现货交易 |
| kraken.future.spot | future | 空头保证金中转站 |
| kraken.future.future | future | 空头合约交易 |
| kraken.close_position.spot | close_position | 子账户平仓 |

kraken的底层配置信息

| 名称 | 用途 |
|----------|----------|
| api_key + api_secret | API 请求签名认证 |
| account_public_id | Spot 账户间转账时标识目标账户 |

kraken.future的底层配置信息

| 名称 | 用途 |
|----------|----------|
| api_key + api_secret | API 请求签名认证 |
| uid | Future 账户间转账时标识目标账户 |

