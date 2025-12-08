# 转账

文档描述了一个多账户资产管理模块，对多个交易所的多个账户的多个钱包进行统一封装，提供：

* **转账功能**：账户间、钱包间、跨交易所的资产转移
* **余额查询**：统一的余额获取接口

## 账户体系

转账系统基于底层的账户体系来工作，统一的账户体系，屏蔽了不同交易所，不同钱包类型的差异。转账系统在使用的时候，不需要关心底层的这些差异

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

## 获取资产

### 🔥 统一接口

* 可以获取不同交易所，不同子账户，不同钱包内的资产

!!! note "注意"

    * 对于不同交易所，不同子账户，不同钱包内的资产可能会返回负数，负数的含义会不一样
    * 对于kraken，资产返回负数表示该资产支持借贷，并且目前自己的资产已经用完，开始使用借贷的部分，会返回负数，这时候该资产是无法转出的
    * 对于kraken future，usd返回负数表示表示有欠款，未实现盈亏还没有结算，这时候是无法转usd出来的
    * 函数使用之前，`AccountType`要预先配置好对应的`api_key & api_secret`
    * 该函数只会查询并返回`symbols`对应的资产数量，没有定义在`symbols`中的资产不会返回

```python
async def get_balances(
    exchange: Exchange,          # BINANCE / KRAKEN / KRAKEN_FUTURE
    account: AccountType,        # main / trade / future / close_position
    symbols: List[Currency] = [],
) -> Result[Dict[Currency, float], Any]
```

### 示例

**获取不同交易所，不同子账户，不同钱包内的资产**


```python
from hftpy.exchange.multi_account_api import get_balances
from hftpy.common import Exchange, Currency

# 查询 Binance trade 账户的 USDC/USDT 余额
result = await get_balances(
    Exchange.BINANCE, 
    "trade", 
    [Currency.USDC, Currency.USDT]
)

# 查询 Kraken trade 账户的多个币种余额
result = await get_balances(
    Exchange.KRAKEN, 
    "trade", 
    [Currency.USD, Currency.BTC, Currency.ETH]
)

# 查询 Kraken Future main 账户余额
result = await get_balances(
    Exchange.KRAKEN_FUTURE, 
    "main", 
    [Currency.USD, Currency.USDT]
)

# 处理结果
if result.success:
    for currency, balance in result.data.items():
        print(f"{currency.name}: {balance}")
else:
    print(f"查询失败: {result.error}")
```

## 转账

### 转账类型分类

* 有三大类的转账

```
转账类型
├── 同交易所转账
│   ├── 类型一：同钱包-跨账户
│   │   ├── binance_spot_transfer      # Binance Spot 子账户间
│   │   ├── kraken_spot_transfer       # Kraken Spot 子账户间
│   │   └── kraken_future_transfer     # Kraken Future 子账户间
│   └── 类型二：同账户-跨钱包
│       ├── kraken_spot_to_future_transfer  # Kraken Spot → Kraken Future
│       └── kraken_future_to_spot_transfer  # Kraken Future → Kraken Spot
└── 类型三：跨交易所转账
    ├── kraken_to_binance_withdraw     # Kraken → Binance
    └── binance_to_kraken_withdraw     # Binance → Kraken
```

**三种转账维度**

| 维度 | 说明 | 示例 |
|------|------|------|
| 类型一：同钱包-跨账户 | 同类型钱包在不同账户间转账 | main.spot → trade.spot |
| 类型二：同账户-跨钱包 | 同一账户内 Spot 与 Future 间转账 | main.spot → main.future |
| 类型三：跨交易所转账 | 不同交易所间提币 | kraken.main.spot → binance.trade.spot |

### 权限与约束

| 转账类型 | API 使用权限 | 约束条件 |
|----------|--------------|----------|
| Binance Spot 账户间 | 必须用主账户 API | 需要配置 email |
| Kraken Spot 账户间 | 必须用主账户 API | 需要配置 account_public_id  |
| Kraken Future 账户间 | 必须用主账户 API | 需要配置 uid |
| Kraken Spot → Future | 用对应账户自己的 API | 同一子账户内 |
| Kraken Future → Spot | 用对应账户自己的 API | 同一子账户内 |
| Kraken → Binance | Kraken 主账户 API | 仅从 main 发起 |
| Binance → Kraken | Binance 主账户 API | main → main |

### 🔥 统一接口

* 可以在不同交易所，不同子账户，不同钱包内进行转账

!!! note "注意"

    * 函数使用之前，`AccountType`要预先配置好对应的`api_key & api_secret`
    * 对于不同子账户之间进行转账，参考**权限与约束**中的**约束条件**进行配置

```python
async def transfer(
    from_exchange: Exchange,
    from_account: AccountType,
    to_exchange: Exchange,
    to_account: AccountType,
    symbol: Currency,
    amount: float,
) -> Result[Dict, Any]:
```

## 转账案例

### Binance Spot 账户间

* 可以在任意子账户的spot之间相互转账
* 只能使用主账户的api key和api secret来执行转账
* 底层需要设置每个账户的email信息, 如果不指定from_email或者to_email, 也就是设置为None, 默认为主账户
* 我们在底层已经配置了account和email映射, 可以直接使用

```python
# 方式一: 使用 底层转账 函数
result = await binance_spot_transfer(
    symbol=Currency.USDT,
    amount=10.0,
    from_account="trade",
    to_account="main"
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.BINANCE,
    from_account="trade",
    to_exchange=Exchange.BINANCE,
    to_account="main",
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```

### Kraken Spot 账户间

* 可以在任意子账户的spot之间相互转账
* 只能使用主账户的api key和api secret来执行转账
* 底层需要设置每个账户的public account id信息来标识每个账户
* 我们在底层已经配置了account和public account id映射, 可以直接使用

```python
# 方式一: 使用 底层转账 函数
result = await kraken_spot_transfer(
    symbol=Currency.USDT,
    amount=1.0,
    from_account="future",
    to_account="close_position"
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.KRAKEN,
    from_account="future",
    to_exchange=Exchange.KRAKEN,
    to_account="close_position",
    symbol=Currency.USDT,
    amount=1.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```

### Kraken Future 账户间

* 可以在任意子账户的future之间相互转账
* 只能使用主账户的api key和api secret来执行转账
* 底层需要设置每个账户的uid信息来标识每个账户
* 目前只有 main和 future 账户有配置uid, trade 和 close_position 暂无

```python
# 方式一: 使用 底层转账 函数
result = await kraken_future_transfer(
    symbol=Currency.USDT,
    amount=10.0,
    from_account="main",
    to_account="future"
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.KRAKEN_FUTURE,
    from_account="main",
    to_exchange=Exchange.KRAKEN_FUTURE,
    to_account="future",
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```

### Kraken Spot → Future

* 使用对应spot账户的api key 和api secret
* 同一个账户内，从 spot -> future

```python
# 方式一: 使用 底层转账 函数
result = await kraken_spot_to_future_transfer(
    symbol=Currency.USDT,
    amount=10.0,
    account="main"
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.KRAKEN,
    from_account="main",
    to_exchange=Exchange.KRAKEN_FUTURE,
    to_account="main",
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```

### Kraken Future → Spot

* 使用对应future账户的api key 和api secret
* 同一个账户内，从 future -> spot

```python
# 方式一: 使用 底层转账 函数
result = await kraken_future_to_spot_transfer(
    symbol=Currency.USDT,
    amount=10.0,
    account="main"
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")   

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.KRAKEN_FUTURE,
    from_account="main",
    to_exchange=Exchange.KRAKEN,
    to_account="main",
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```

### Kraken → Binance

* 底层使用 Kraken 提现接口将指定币种的资金从账户中提取，发送至预设的提现地址（通过 address 和 key 指定）。地址必须已在 Kraken 后台添加并命名（key）且通过验证。
* 该API的功能：从 Kraken 主账户提币到 Binance 账户，可以是主账户，也可以是子账户
* 目标address对应一个目标账户，key对应一个币种，例如：我们要转账到binance子账户，子账户有一个address，一个key对应一个币种
    * address: `0x....a651`
    * key: `sub1-usdc`, 币种: usdc
    * key: `sub1-usdt`, 币种: usdt
* 所有的配置提前已经设置好，只需要调用提现接口即可，不需要关心address和key的配置细节

```python
# 方式一: 使用 底层转账 函数
result = await kraken_to_binance_withdraw(
    symbol=Currency.USDT,
    amount=10.0,
    to_account="trade"
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.KRAKEN,
    from_account="main",
    to_exchange=Exchange.BINANCE,
    to_account="trade",
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```

### Binance → Kraken

* 从 Binance 主账户提币到 Kraken 主账户。不能转到Kraken的子账户
* 转所有的币，公用一个address
* 所有的配置提前已经设置好，只需要调用提现接口即可，不需要关心address的配置细节

```python
# 方式一: 使用 底层转账 函数
result = await binance_to_kraken_withdraw(
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")

# 方式二: 使用 统一转账 函数
result = await transfer(
    from_exchange=Exchange.BINANCE,
    from_account="main",
    to_exchange=Exchange.KRAKEN,
    to_account="main",
    symbol=Currency.USDT,
    amount=10.0
)
if result.success:
    logger.info(f"转账成功: {result.data}")
else:
    logger.warning(f"转账失败: {result.error}")
```