# 转账任务

## 业务场景

我们有一个账户，里面跑了多个子策略，每个子策略会占用一定的资金。某个逻辑会在某一时刻有资金转出的需求，但是这时候资金往往是被占用的，无法立即执行资金转出操作。我们需要在未来的某个时间点（资金可用的时候）执行转账操作。也就是转账需求的提出，和转账需求的执行是分开的。这个模块就是为了解决这个问题。

只要有资金转出的需求，不需要关心当前资金是否被占用，只需要**提交一个转账任务**就结束了。会有一个执行器，在合理的时间点遍历所有的转账任务并执行，这在逻辑实现上会变得很简洁。


## 案例

!!! note "注意"

    * 使用[统一API：transfer](transfer_api.md)进行转账
    * **执行转账任务**的时间点可以是撤挂单的间隙，如果转账失败应该重试，直到任务成功为止
    * **重复转账的问题**：同样的转账任务，也就是`TransferTaskKey`相同的任务，可以重复添加，并不会生成多条转账任务，在任务成功执行前，只会有一条转账任务
    * 这里描述的是单步转账的任务，也就是成功执行一次`A->B`转账操作就结束了。另外还有链式转账任务`A->B->C->D`，链式转账任务依赖于单步转账任务。我们通常的做法是成功执行一次单步`A->B`转账任务后，再创建后续的转账任务链`B->C->D`，有专门的**转账任务执行器**来执行后续的链式转账操作。后续步骤的到账时间未知，涉及到链上提款时间会更久，所以需要一个专门的转账任务执行器来执行后续的转账操作，直到资金成功转入最终的目标账户
    * 在创建转账任务的时候，有一个附加字段`extra`，作用是转账成功后，可以根据这个字段执行自定义的后续业务逻辑

**数据结构**

```python
class TransferTaskKey(NamedTuple):
    """转账任务Key"""
    from_exchange: Exchange
    from_account: str
    to_exchange: Exchange
    to_account: str
    currency: Currency


class TransferTaskValue(NamedTuple):
    """转账任务Value"""
    amount: float
    timestamp: int
    extra: Optional[str] = None  # 附加信息
```

**使用示例**

```python
import time
from hftpy.common import Exchange, Currency, TransferTaskManager, TransferTaskKey
from hftpy.exchange.multi_account_api import transfer

# 创建管理器，设置默认过期时间120秒
manager = TransferTaskManager(default_expire_seconds=120.0)

# ========== 生产者：子策略提交转账需求 ==========

# 子策略A：需要从 Kraken trade 转 1000 USDC 到 Kraken main
# 如果任务已存在则跳过
manager.add_task(
    from_exchange=Exchange.KRAKEN,
    from_account="trade",
    to_exchange=Exchange.KRAKEN,
    to_account="main",
    currency=Currency.USDC,
    amount=1000.0
    extra=""
)

# 子策略B：需要从 Kraken trade 转 500 USD 到 kraken close_position
# 如果任务已存在则跳过，如果120秒内已成功转账过则不重复提交
key_b = TransferTaskKey(
    from_exchange=Exchange.KRAKEN,
    from_account="trade",
    to_exchange=Exchange.KRAKEN,
    to_account="close_position",
    currency=Currency.USD
)
last_success = manager.get_last_success_time(key_b)
current_time = int(time.time() * 1_000_000)
if last_success is None or current_time - last_success > 120 * 1_000_000:
    manager.add_task(
        from_exchange=key_b.from_exchange,
        from_account=key_b.from_account,
        to_exchange=key_b.to_exchange,
        to_account=key_b.to_account,
        currency=key_b.currency,
        amount=500.0,
        extra=""
    )

# ========== 消费者：执行器遍历并执行 ==========

# 获取任务快照（协程安全）
for key, value in manager.get_all_tasks():
    # 检查条件：资金是否充足
    if not can_transfer(key.from_exchange, key.from_account, key.currency, value.amount):
        continue
    
    # 执行转账
    result = await transfer(
        from_exchange=key.from_exchange,
        from_account=key.from_account,
        to_exchange=key.to_exchange,
        to_account=key.to_account,
        symbol=key.currency,
        amount=value.amount
    )
    
    if result.success:
        # 转账成功，删除任务
        manager.remove_task(key)

        # 记录成功转账的时间
        manager.mark_success(key)

        if value.extra = "..."
            # 可以执行后续逻辑
            pass
    # 转账失败，保留任务，下次重试

# ========== 定期清理过期任务 ==========
cleaned = manager.clean_expired_tasks()
```

