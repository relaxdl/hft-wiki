# 共享内存基础

## 设计原则

* 使用system V的共享内存
* 一个进程负责写(Producer)某一块共享内存，比如某个交易所某个交易对的Ticker数据，多个进程读
* 写进程不能有锁，新的数据来了之后随时可以写入，只保留当下时刻的最新一条记录，不保留历史
* 读进程(Consumer)不能有锁，随时可以读到数据。读进程不要求必须读到最新的数据，只要读到最近一条完整的数据就可以。比如：写进程正在写的数据不需要读，因为数据写了一半，没写完不完整。只需要读刚写完的最新一条完整数据就可以
    * 写进程总是在更新下一个位置，写完才更新索引
    * 读进程只需读取writer已完成更新的最新索引即可
* 读写都要追求极致的性能
* 原子索引使用acquire/release语义(补充：atomic在多线程中可以用来同步，在多进程中没有效果)
    * 写进程写索引(writer_index)，使用`memory_order_release`，表示之前所有对数据的写入对读进程可见(多进程中没有效果，保留也不影响性能)
    * 读进程读索引(writer_index)，使用`memory_order_acquire`，保证读进程能正确看到索引前的所有写入操作(多进程中没有效果，保留也不影响性能)
* 考虑CPU Cache Line和False Sharing
    * `alignas(64)`和`padding`确保`Ticker`等结构体对齐CPU缓存行，提高读取效率
    * 消除False Sharing
        * 每个slot的Ticker数据单元正好占用一个或多个cache line
        * writer和reader进程不会共享同一Cache line
        * 大幅减少L1/L2 cache的失效，提升吞吐量和降低延迟
    * 提高数据访问局部性：读数据时一次性加载完整cache line，提高内存吞吐率
* 使用环形缓冲区(Ring Buffer)，**假设缓冲区足够大**，比如：尺寸是512，可以保证读进程正在读一个slot的时候，写进程不会写完一圈又写回来。这样能保证数据的完整性。通过这个假设，省去了读的完整性校验步骤
    * 数据损坏的本质原因是：当读进程正在读取某个slot数据时，写进程恰好正在同时写入该slot的数据，导致读进程可能读到部分新数据和部分旧数据，出现数据撕裂(Data Tearing)

## System V Shared Memory

SysV 共享内存使用步骤非常固定

1. **`ftok`**（可选）- 生成唯一 key
2. **`shmget`** - 创建/获取共享内存段
3. **`shmat`** - 将共享内存附加到进程地址空间
4. **读写操作** - 直接访问共享内存
5. **`shmdt`** - 分离共享内存
6. **`shmctl`** - 删除共享内存

## System V API的封装

* 对system V共享内存API的封装

```c++
生成共享内存的键值 key_t
key_t generateShmKey(const std::string &path, char projId = 'a');

创建/获取共享内存段
int createShm(key_t key, size_t size, bool overwrite = false);
int createShm(const std::string &path, size_t size, bool overwrite = false);

将共享内存附加到进程地址空间
void *attachShm(key_t key, size_t size, bool readOnly = false);
void *attachShm(const std::string &path, size_t size, bool readOnly = false);

分离共享内存
int detachShm(void *shmPtr);

删除共享内存
int deleteShm(key_t key, size_t size = 0);
int deleteShm(const std::string &path, size_t size = 0);

通过共享内存 ID 删除共享内存空间
int deleteShmById(int shmid);
```

## System V API最佳实践

1. **`generateShmKey`** - 生成共享内存的键值
2. **`createShm`** - 创建/获取共享内存段
3. **`attachShm`** - 将共享内存附加到进程地址空间

经过上面三个步骤后，我们可以用`SharedMemoryTickerData *shmPtr_`这个指针，来操作这块共享内存

!!! warning "注意"

    * 在实盘中，我们attach的内存统一都是Ring Buffer封装后的数据结构
    * 虽然封装好的共享内存API提供了overwrite的功能来重新创建一块新的共享内存，但是在实际使用的时候，**我们永远不会删除已有的一块共享内存再去重新创建它**
    * 目前唯一需要将一块已有的共享内存删除再重新创建的业务场景就是修改了底层的数据结构，在底层基础数据结构稳定的情况下，这个需求非常少。及时有必须要改动的场景，升级的时候，重启服务器，重新刷新整个共享内存即可。这样可以极大简化上层业务逻辑时间

```c++ hl_lines="14 21 28 42"
class SharedMemoryTickerRingBuffer
{
public:
    /**
     * 构造函数，初始化共享内存 RingBuffer
     *
     * @param exchange 交易所（枚举类型，如 BINANCE, OKX）
     * @param symbol 交易对（枚举类型，如 BTC_USDT, ETH_USDT）
     * @param overwrite 是否覆盖已有的共享内存（默认 false）
     */
    SharedMemoryTickerRingBuffer(const Exchange exchange, const CurrencyPair symbol, bool overwrite = false)
    {
        shmFilePath_ = PathManager::getShmFilePath(exchange, Channel::TICKER, symbol);
        shmKey_ = generateShmKey(shmFilePath_);
        if (shmKey_ < 0)
        {
            perror("generateShmKey error");
            exit(1);
        }

        shmid_ = createShm(shmKey_, sizeof(SharedMemoryTickerData), overwrite);
        if (shmid_ < 0)
        {
            perror("createShm error");
            exit(1);
        }

        shmPtr_ = static_cast<SharedMemoryTickerData *>(attachShm(shmKey_, sizeof(SharedMemoryTickerData), false));
        if (!shmPtr_)
        {
            perror("attachShm error");
            exit(1);
        }
    }

    // ...

private:
    std::string shmFilePath_;
    key_t shmKey_;
    int shmid_;
    SharedMemoryTickerData *shmPtr_ = nullptr;
};
```

## 具体的数据结构

以Ticker为案例介绍，其它类型的数据实现方式和对外的接口都是完全一致的

* Ticker：基础数据结构
* SharedMemoryTickerData：共享内存布局Ring Buffer
* SharedMemoryTickerRingBuffer：Ring Buffer管理类，提供操作接口
* 🔥 SharedMemoryTicker：全局Ring Buffer管理器，上层使用者最终只需要和这个类交互

### 基础数据结构

* Ticker
* 按照Cache Line对齐的基础数据结构

### 共享内存布局Ring Buffer

* SharedMemoryTickerData
* 共享内存的数据结构，就是Ticker的Ring Buffer
* 一个交易所，一个交易对的Ticker

### Ring Buffer管理类

* SharedMemoryTickerRingBuffer
* 单个Ring Buffer的管理器，**管理一个特定交易对的Ticker的共享内存**，负责操作SharedMemoryTickerData，对外给出操作接口
* 提供读写接口，支持零拷贝操作
* RAII 风格，自动管理资源生命周期


!!! warning "注意"

    * 在使用这个类的时候要注意，每构造一个实例，会自动attach到已存在的共享内存。一个进程如果创建多个实例，会attach多次到同一块内存，这不是我们期望的行为。
    * 这个类比较底层，在实际使用的时候，不需要直接访问这个类，直接使用上层的`SharedMemoryTicker`，会自动管理多个交易对的`SharedMemoryTickerData`，同时会避免attach到同一块内存多次这样的行为

#### API

* 对外暴露的API非常简单，只有如下4个
* 支持数据的读取
* 支持数据的写入（拷贝和零拷贝）

```c++
/**
 * 构造函数，初始化共享内存 RingBuffer
 *
 * @param exchange 交易所（枚举类型，如 BINANCE, OKX）
 * @param symbol 交易对（枚举类型，如 BTC_USDT, ETH_USDT）
 * @param overwrite 是否覆盖已有的共享内存（默认 false）
 */
SharedMemoryTickerRingBuffer(const Exchange exchange, const CurrencyPair symbol, bool overwrite = false);

inline Ticker *get();

/**
 * 向 RingBuffer 写入新的 Ticker 数据
 *
 * @param ticker 需要写入的 Ticker 结构
 */
inline void write(const Ticker &ticker);

/**
 * 获取可写入的 Ticker 指针（避免拷贝）
 *
 * @return 指向可写入位置的指针
 */
inline Ticker *getWritable();

/**
 * 提交写入操作，更新 RingBuffer 的写索引
 */
inline void commit();
```

#### 案例1：生产者进程（拷贝式写入）

```c++
SharedMemoryTickerRingBuffer buffer(
    Exchange::BINANCE, 
    CurrencyPair::BTC_USDT
);

Ticker ticker;
ticker.askPrice = 50000.0 + (rand() % 100);
ticker.askVolume = 1.5;
ticker.bidPrice = 49999.0 + (rand() % 100);
ticker.bidVolume = 2.0;

// 直接写入（有拷贝），会把ticker拷贝到底层的Ring Buffer对应的slot
buffer.write(ticker);
```

#### 案例2：生产者进程（零拷贝优化版）

* `buffer.getWritable()`和`buffer.commit()`操作必须一一对应

```c++
SharedMemoryTickerRingBuffer buffer(
    Exchange::BINANCE, 
    CurrencyPair::BTC_USDT
);

// 获取可写位置的指针
Ticker* ptr = buffer.getWritable();

// 更新ticker
// 直接在共享内存中填充数据（零拷贝！）
ptr->askPrice = 50000.0 + (rand() % 100);
ptr->askVolume = 1.5;
ptr->bidPrice = 49999.0 + (rand() % 100);
ptr->bidVolume = 2.0;

// 提交写入（更新索引）
buffer.commit();
```

#### 案例3：消费者进程（数据读取方）

```c++
SharedMemoryTickerRingBuffer buffer(
    Exchange::BINANCE, 
    CurrencyPair::BTC_USDT
);

// 获取最新数据（零拷贝，直接返回指针）
Ticker* ticker = buffer.get();
std::cout << "Ask=" << ticker->askPrice << " "
          << "Bid=" << ticker->bidPrice << " "
          << "Time=" << ticker->timestamp << std::endl;
```

### 全局Ring Buffer管理器

* SharedMemoryTicker
* 全局统一的 Ticker 共享内存管理器，管理所有交易所、所有交易对的 Ticker 共享内存
* 提供全局静态接口，无需手动管理 RingBuffer 实例
* 懒加载机制：首次访问时自动创建共享内存
* 自动路由：根据 exchange 和 symbol 自动找到对应的 RingBuffer


#### API

* 对外暴露的API非常简单，只有如下5个
* 支持数据的读取（拷贝和零拷贝）
* 支持数据的写入（拷贝和零拷贝）

```c++
/**
 * 写入 Ticker 数据
 *
 * @param ticker 需要写入的 Ticker 结构体
 */
static inline void write(const Ticker &ticker);

/**
 * 获取最新的 Ticker 数据
 *
 * @param exchange 交易所（枚举类型，如 BINANCE, OKX）
 * @param symbol 交易对（枚举类型，如 BTC_USDT, ETH_USDT）
 * @return 指向最新 Ticker 数据的指针
 */
static inline Ticker *get(const Exchange exchange, const CurrencyPair symbol);

/**
 * 获取最新的 Ticker拷贝 数据
 *
 * @param exchange 交易所（枚举类型，如 BINANCE, OKX）
 * @param symbol 交易对（枚举类型，如 BTC_USDT, ETH_USDT）
 * @return 最新 Ticker 的拷贝
 */
static inline Ticker getCopy(const Exchange exchange, const CurrencyPair symbol);

/**
 * 获取可写入的 Ticker 指针，避免拷贝
 *
 * @param exchange 交易所（枚举类型，如 BINANCE, OKX）
 * @param symbol 交易对（枚举类型，如 BTC_USDT, ETH_USDT）
 * @return 指向可写入位置的指针
 */
static inline Ticker *getWritable(const Exchange exchange, const CurrencyPair symbol);

/**
 * 提交写入的 Ticker 数据
 *
 * @param exchange 交易所（枚举类型，如 BINANCE, OKX）
 * @param symbol 交易对（枚举类型，如 BTC_USDT, ETH_USDT）
 */
static inline void commit(const Exchange exchange, const CurrencyPair symbol);
```

#### 🔥 案例1：生产者进程（拷贝式写入）

```c++
// 解析 JSON 到 Ticker
Ticker ticker = parseFromJson(json);

// 直接写入（最简单）
// 自动根据 ticker.exchange 和 ticker.symbol 路由到对应的共享内存
ShareMemoryTicker::write(ticker);  
```

#### 🔥 案例2：生产者进程（零拷贝优化）


```c++
// 交易所和交易对
Exchange exchange = Exchange::BINANCE;
CurrencyPair symbol = CurrencyPair::BTC_USDT;

// 获取共享内存中的可写位置
Ticker* ptr = ShareMemoryTicker::getWritable(exchange, symbol);

// 直接在共享内存中解析和填充数据（避免临时对象）
auto doc = parser.iterate(json);
ptr->exchange = exchange;
ptr->symbol = symbol;
ptr->timestamp = doc["E"].get_int64();
ptr->localTimestamp = getCurrentTimestamp();
ptr->askPrice = doc["a"].get_double();
ptr->askVolume = doc["A"].get_double();
ptr->bidPrice = doc["b"].get_double();
ptr->bidVolume = doc["B"].get_double();
    
// 提交写入（更新索引，让消费者可见）
ShareMemoryTicker::commit(exchange, symbol);
```

#### 🔥 案例3：消费者进程（数据读取方）

```c++
// 交易所和交易对
Exchange exchange = Exchange::BINANCE;
CurrencyPair symbol = CurrencyPair::BTC_USDT;

// 读取最新的 Ticker 数据
Ticker* ticker = ShareMemoryTicker::get(exchange, symbol);

std::cout << "Ask=" << ticker->askPrice << " "
          << "Bid=" << ticker->bidPrice << " "
          << "Spread=" << (ticker->askPrice - ticker->bidPrice)
          << std::endl;
```