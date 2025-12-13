# gateway的资产管理


* 网关模块负责管理和同步资产。将最新的资产信息写入共享内存，其它进程可以从共享内存中读取最新的资产信息
* 共享内存中所有的信息，只能有一个写入者，所以**网关模块是资产的唯一维护者，其它所有的模块只能读取，不能修改资产**
* 支持虚拟账户，详见 [共享内存资产&策略资产](shm_balance.md)

## 如何维护资产

* **初始化**：系统初始化的时候，通过REST API获取资产余额 → 初始化资产
* **成交**：每次有成交发生的时候，本地计算余额变化 → 更新到共享内存
* **定时同步**：每60秒 REST API 校验并修正，有具体的校验和修正逻辑
* **IPC命令**：外部指令调整资金 → 调用虚拟账户相关的API来分配&管理资金

## 如何读取资产

**通过API直接从共享内存中读取资产信息**

### 资产

**简单的资产逻辑，没有虚拟子账户的概念**

```python
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

### 策略资产

**模拟虚拟子账户的逻辑，每个虚拟子账户对一个子策略**

```python
from hftpy.common import StrategyBalanceManager
import hft

# 创建管理器（Kraken交易所，8个策略槽位）
manager = StrategyBalanceManager(hft.Exchange.KRAKEN, strategy_count=8)
total = manager.get_balance(hft.Currency.USDC, index=0)      # 总资金
available = manager.get_balance(hft.Currency.USDC, index=1)  # 主策略（可用）
strategy_a = manager.get_balance(hft.Currency.USDC, index=2) # 策略A
strategy_b = manager.get_balance(hft.Currency.USDC, index=3) # 策略B

print(f"USDC: total={total}, available={available}, strategy_a={strategy_a}, strategy_b={strategy_b}")

```

## IPC命令

**策略和网关在不同的进程，通过RPC的方式进行资金操作**

### 子策略申请或者归还资金

* 调用allocate或者release来申请或者归还资金

=== "JSON"

    ```json
    // 申请资金
    {
        "strategy": "a",          // 策略名称
        "cmd": "allocate",        // 申请资金
        "currency": "usd",        // 币种
        "amount": 100,            // 申请金额
        "index": 2                // 子策略索引
    }

    // 归还资金
    {
        "strategy": "a",          // 策略名称
        "cmd": "release",         // 归还资金
        "currency": "usd",        // 币种
        "amount": 100,            // 归还金额
        "index": 2                // 子策略索引
    }
    ```

=== "Python"

    ```python
    from hftpy.ipc.ipc_command_system import IPCCommandSystem
    from hftpy.common import KRAKEN_STRATEGY_BALANCE_CMD
    from hftpy.common.kraken_gateway_proto import kraken_strategy_balance

    # 初始化 IPC 命令系统
    command_system = IPCCommandSystem("my_strategy")
    command_system.start()

    # 申请资金
    payload = kraken_strategy_balance(
        strategy="a",
        cmd="allocate",
        currency="usd",
        index=2,
        amount=100
    )
    await command_system.send_command("kraken_gateway", KRAKEN_STRATEGY_BALANCE_CMD, payload)

    # 归还资金
    payload = kraken_strategy_balance(
        strategy="a",
        cmd="release",
        currency="usd",
        index=2,
        amount=100
    )
    await command_system.send_command("kraken_gateway", KRAKEN_STRATEGY_BALANCE_CMD, payload)
    ```

### 转账成功

* 成功转入或者转出后，调用adjust通知网关调整总资产

=== "JSON"

    ```json
    // 转账：资金转出
    {
        "strategy": "a",       // 策略名称
        "cmd": "adjust",       // 命令类型
        "currency": "usd",     // 币种
        "amount": -1000,       // 调整金额（负数表示转出）
        "index": 0             // 索引（0=总资金）
    }
    ```

=== "Python"

    ```python
    from hftpy.ipc.ipc_command_system import IPCCommandSystem
    from hftpy.common import KRAKEN_STRATEGY_BALANCE_CMD
    from hftpy.common.kraken_gateway_proto import kraken_strategy_balance

    # 初始化 IPC 命令系统
    command_system = IPCCommandSystem("my_strategy")
    command_system.start()

    # 转账成功后通知网关调整总资产
    payload = kraken_strategy_balance(
        strategy="a",
        cmd="adjust",
        currency="usd",
        delta=-1000,  # 负数表示转出
        index=0       # 0=总资金
    )
    await command_system.send_command("kraken_gateway", KRAKEN_STRATEGY_BALANCE_CMD, payload)
    ```