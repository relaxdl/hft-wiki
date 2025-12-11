# 共享内存资产&策略资产

## 核心需求

* 主要解决同一个账户内资金动态分配的问题
* 支持一个账户下多个子策略的资金分配，一个账户内的资金分成多份，给多个子策略使用。每个子策略对应一个**虚拟账户**，虚拟账户内的钱是这个子策略的最大可用资金，不需要用完
* 支持资金转出和转出这样的业务场景
* 发现和交易所资金不一致的时候，有恢复策略
* 支持跨进程同步，支持借贷

## 资产

如果一个账户内的资产只对应一个策略，不需要做资金划分，用如下简单的方式就可以跨进程的操作资产

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

## 策略资产

* 模拟虚拟子账户的逻辑，每个虚拟子账户对一个子策略
* 最多支持63个子策略(1个主策略/钱包 + 62个其他策略)
    * strategy[0] = 总资金
    * strategy[1] = 主策略（也可以理解成钱包或者资金池）
    * strategy[2..N] = 其它策略资金（配额）
* 在使用之前要先初始化，具体的策略数量上层逻辑自己确定，只有`strategies[0..strategyCount]`范围内的数据保证有效

### 结构

```
┌─────────────────────────────────────────────────────────────────┐
│                      StrategyBalance                            │
│  Exchange: KRAKEN    Currency: USDC    strategyCount: 5         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   strategies[64] 数组                                           │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ [0] 总资金: 10000.0  ◄── 账户真实余额                      │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ [1] 主策略: 4000.0   ◄── 资金池，其他策略从这里申请          │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ [2] 策略A:  2000.0   ◄── 子策略占用                       │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ [3] 策略B:  3000.0   ◄── 子策略占用                       │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ [4] 策略C:  1000.0   ◄── 子策略占用                       │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │ [5..63] 未使用                                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   约束: strategies[0] = strategies[1] + strategies[2] + ...     │
│         10000 = 4000 + 2000 + 3000 + 1000 ✓                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 资金流动

* **恒等式**：`strategies[0] = sum(strategies[1:strategyCount])`
* **申请资金**：`strategies[1] -= amount;  strategies[i] += amount;`
* **归还资金**：`strategies[i] -= amount;  strategies[1] += amount;`

```
                    ┌──────────────────────┐
                    │   [0] 总资金 10000    │
                    │   (账户真实余额)       │
                    └──────────────────────┘
                              │
                              │ 分配
                              ▼
        ┌─────────────────────────────────────────────┐
        │             [1] 主策略 4000                  │
        │             (资金池/未分配)                   │
        └─────────────────────────────────────────────┘
                │             │             │
         申请资金│       申请资金│      申请资金│
                ▼             ▼             ▼
        ┌───────────┐ ┌───────────┐ ┌───────────┐
        │[2] 策略A   │ │[3] 策略B  │  │[4] 策略C  │
        │   2000    │ │   3000    │ │   1000    │
        └───────────┘ └───────────┘ └───────────┘
                │             │             │
         归还资金│       归还资金│      归还资金│
                └─────────────┼─────────────┘
                              ▼
                    ┌──────────────────────┐
                    │   [1] 主策略 (资金池)  │
                    └──────────────────────┘
```

### API

所有涉及资金操作的API调用之后，会自动再平衡，保证**恒等式**的成立

| 方法 | 说明 |
|------|------|
| `reset(currency, balance)` | 重置：总资金=主策略=balance，子策略=0 |
| `set(currency, amount, index=0)` | 设置某索引的值，通常不需要直接使用这个API |
| `adjust(currency, delta, index=0)` | 增减某索引的值（+/-），通常只会操作总资金 |
| `allocate(currency, index, amount)` | 子策略申请资金 |
| `release(currency, index, amount)` | 子策略归还资金 |
| `get(currency)` | 获取完整 StrategyBalance |
| `get_balance(currency, index=0)` | 获取某索引的资金，默认返回总资金 |


### 业务场景

#### 系统启动

* 调用`reset`初始化资产

```python
from hftpy.common import StrategyBalanceManager
import hft

# 创建管理器（Kraken交易所，8个策略槽位）
manager = StrategyBalanceManager(hft.Exchange.KRAKEN, strategy_count=8)

# 获取账户余额（从交易所API）
usdc_balance = 10000.0
usd_balance = 5000.0

# 初始化各币种资产
manager.reset(hft.Currency.USDC, usdc_balance)
manager.reset(hft.Currency.USD, usd_balance)
```

#### 资产买卖

* **卖出一个资产，或者买入一个资产**
* 本质是总资金的变化，调用`adjust`调整总资产

```python
# 买入：买入usdc/usd
manager.adjust(hft.Currency.USD, -1000.0)   # USD 减少
manager.adjust(hft.Currency.USDC, 1000.0)   # USDC 增加

# 卖出：卖出usdc/usd
manager.adjust(hft.Currency.USDC, -500.0)   # USDC 减少
manager.adjust(hft.Currency.USD, 500.0)     # USD 增加
```

#### 子策略申请或者归还资金

* 调用`allocate`或者`release`来申请或者归还资金

```python
# 策略A（index=2）申请 2000 USDC
manager.allocate(hft.Currency.USDC, index=2, amount=2000.0)

# 策略B（index=3）申请 3000 USDC
manager.allocate(hft.Currency.USDC, index=3, amount=3000.0)

# 策略A 归还 500 USDC
manager.release(hft.Currency.USDC, index=2, amount=500.0)

# 查询各策略资金
total = manager.get_balance(hft.Currency.USDC, index=0)      # 总资金
available = manager.get_balance(hft.Currency.USDC, index=1)  # 主策略（可用）
strategy_a = manager.get_balance(hft.Currency.USDC, index=2) # 策略A
strategy_b = manager.get_balance(hft.Currency.USDC, index=3) # 策略B
```

#### 转账成功

* 成功转入或者转出，本质上和买入和卖出一个资产是一样的，调用`adjust`调整总资产

```python
# 转入 1000 USDC（从其他交易所转入）
manager.adjust(hft.Currency.USDC, 1000.0)

# 转出 500 USDC（转到其他交易所）
manager.adjust(hft.Currency.USDC, -500.0)
```

#### 错误恢复

* **错误恢复的逻辑**：当发现本地的资产和交易所不一致的时候，调整总资产，子策略的资产不动，同时调整主策略（钱包）的余额，保证恒等式成立
* 注意：如果资金修复后，**主策略资金 + 借贷额度 < 0**，表示资金错误是无法修复的，这时候的简单处理方式是`reset`资产，让每个子策略重新申请自己需要的资产

```python
# 从交易所获取真实余额
real_balance = 从交易所获取的资产

# 修正总资产（set会自动refresh，重算主策略）
manager.set(hft.Currency.USDC, real_balance)

# 检查主策略是否为负（考虑借贷额度）
available = manager.get_balance(hft.Currency.USDC, index=1)
credit_limit = 5000.0  # 借贷额度

if available + credit_limit < 0:
    # 资金错误无法修复，需要重置
    logger.error(f"资金错误无法修复，主策略余额为负且超过借贷额度: {available}, 借贷额度: {credit_limit}")
    manager.reset(hft.Currency.USDC, real_balance)
    # 子策略需要重新申请资金...
```

## 借贷

借贷是一个额外的逻辑，账户中存储的资产是不包括借贷额度的。**每种资产的借贷额度配置文件中单独配置**。资产当前的数量+借贷额度，是这个资产当前真正的可用额度。例如：如果一个资产当前的余额是100，借贷额度是50，这个资产的当前可用额度是`100+50=150`；如果一个资产当前的余额是-30，借贷额度是50，这个资产当前可用额度是`-30+50=20`。正常情况下，主策略（资金池）中的资金加上借贷额度，应该是大于等于0的。如果不满足这个条件，表示资金状态出错，需要用`reset`重置资金状态