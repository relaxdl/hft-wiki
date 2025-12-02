# 路径管理

## 介绍

* hft-lib库提供了路径管理的功能，项目中用到的所有路径。由底层C++的`PathManager`来统一管理，方便维护
* 跨进程和跨语言的访问，只要hft root设置的一致，就能保证文件路径的唯一性。例如：我们需要用文件的inode信息生成跨进程共享的share memory key的时候，这个模块可以保证唯一性
* **这个模块只负责返回路径，并不会创建路径和文件**，其它模块负责文件和路径的创建
* 提供了C++和Python两套API给上层应用使用

## 具体实现

C++实现 🔒 *（私有仓库，需要授权访问）*

* [path_manager.h](https://github.com/relaxdl/hft-lib/blob/main/include/hft/util/path_manager.h)
* [path_manager.cpp](https://github.com/relaxdl/hft-lib/blob/main/src/hft/util/path_manager.cpp)

测试代码 🔒 *（私有仓库，需要授权访问）*

* [example_path_manager.cpp](https://github.com/relaxdl/hft-lib/blob/main/example/test/example_path_manager.cpp)
* [test_path_manager.py](https://github.com/relaxdl/hft-lib/blob/main/tools/test_path_manager.py)


## 定义

### hft root

* 是一个目录，后续提到的所有路径都在这个根目录下；系统内所有的文件都保存在一个根目录下，按照不同的用途分类组织
* 如果不设置，系统默认的hft root是：`/tmp/hft`

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 获取默认的 hft root
    std::string root = PathManager::getHftRoot();
    std::cout << "Default root path: " << root << std::endl;

    // 设置自定义的 hft root
    PathManager::setHftRoot("/tmp/hft");
    std::cout << "Root path after setting: " << PathManager::getHftRoot() << std::endl;
    ```

    **▶ 输出：**
    ```
    Default root path: /tmp/hft
    Root path after setting: /tmp/hft
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 获取默认的 hft root
    root = hft.PathManager.getHftRoot()
    print(f"Default root path: {root}")

    # 设置自定义的 hft root
    hft.PathManager.setHftRoot("/tmp/hft")
    print(f"Root path after setting: {hft.PathManager.getHftRoot()}")
    ```

    **▶ 输出：**
    ```
    Default root path: /tmp/hft
    Root path after setting: /tmp/hft
    ```

### tmp file

* 保存一些临时性的文件

```
/tmp/hft/tmp/exchange/dataType/filename
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 字符串版本
    std::string tmpPath1 = PathManager::getTmpFilePath("binance", "trade", "test.csv");
    std::cout << "Tmp file path: " << tmpPath1 << std::endl;

    // 枚举版本
    std::string tmpPath2 = PathManager::getTmpFilePath(Exchange::BINANCE, "ticker", "data.csv");
    std::cout << "Tmp file path (enum): " << tmpPath2 << std::endl;
    ```

    **▶ 输出：**
    ```
    Tmp file path: /tmp/hft/tmp/binance/trade/test.csv
    Tmp file path (enum): /tmp/hft/tmp/binance/ticker/data.csv
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 字符串版本
    tmp_path1 = hft.PathManager.getTmpFilePath("binance", "trade", "test.csv")
    print(f"Tmp file path: {tmp_path1}")

    # 枚举版本
    tmp_path2 = hft.PathManager.getTmpFilePath(hft.Exchange.BINANCE, "ticker", "data.csv")
    print(f"Tmp file path (enum): {tmp_path2}")
    ```

    **▶ 输出：**
    ```
    Tmp file path: /tmp/hft/tmp/binance/trade/test.csv
    Tmp file path (enum): /tmp/hft/tmp/binance/ticker/data.csv
    ```

### model file

* 模型文件

```
/tmp/hft/model/exchange/filename
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 字符串版本
    std::string modelPath1 = PathManager::getModelFilePath("okx", "model_v1.pkl");
    std::cout << "Model file path: " << modelPath1 << std::endl;

    // 枚举版本
    std::string modelPath2 = PathManager::getModelFilePath(Exchange::OKX, "model_v2.pkl");
    std::cout << "Model file path (enum): " << modelPath2 << std::endl;
    ```

    **▶ 输出：**
    ```
    Model file path: /tmp/hft/model/okx/model_v1.pkl
    Model file path (enum): /tmp/hft/model/okx/model_v2.pkl
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 字符串版本
    model_path1 = hft.PathManager.getModelFilePath("okx", "model_v1.pkl")
    print(f"Model file path: {model_path1}")

    # 枚举版本
    model_path2 = hft.PathManager.getModelFilePath(hft.Exchange.OKX, "model_v2.pkl")
    print(f"Model file path (enum): {model_path2}")
    ```

    **▶ 输出：**
    ```
    Model file path: /tmp/hft/model/okx/model_v1.pkl
    Model file path (enum): /tmp/hft/model/okx/model_v2.pkl
    ```

### return file

* 收益率文件，按交易所和交易对区分，对应不同的收益率文件
* 会自动增加`*.csv`后缀

```
/tmp/hft/return/exchange/symbol.csv
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 字符串版本
    std::string returnPath1 = PathManager::getReturnFilePath("binance", "btcusdt");
    std::cout << "Return file path: " << returnPath1 << std::endl;

    // 枚举版本
    std::string returnPath2 = PathManager::getReturnFilePath(Exchange::BINANCE, CurrencyPair::BTC_USDT);
    std::cout << "Return file path (enum): " << returnPath2 << std::endl;
    ```

    **▶ 输出：**
    ```
    Return file path: /tmp/hft/return/binance/btcusdt.csv
    Return file path (enum): /tmp/hft/return/binance/btcusdt.csv
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 字符串版本
    return_path1 = hft.PathManager.getReturnFilePath("binance", "btcusdt")
    print(f"Return file path: {return_path1}")

    # 枚举版本
    return_path2 = hft.PathManager.getReturnFilePath(hft.Exchange.BINANCE, hft.CurrencyPair.BTC_USDT)
    print(f"Return file path (enum): {return_path2}")
    ```

    **▶ 输出：**
    ```
    Return file path: /tmp/hft/return/binance/btcusdt.csv
    Return file path (enum): /tmp/hft/return/binance/btcusdt.csv
    ```

### prob file

* 概率表文件，按交易所，交易对区分，每个交易对买卖分开，对应不同的概率表文件

```
/tmp/hft/prob/exchange/symbol_side.csv
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 字符串版本
    std::string probPath1 = PathManager::getProbFilePath("binance", "ethusdt", "buy");
    std::cout << "Prob file path: " << probPath1 << std::endl;

    // 枚举版本
    std::string probPath2 = PathManager::getProbFilePath(Exchange::BINANCE, CurrencyPair::ETH_USDT, Side::BUY);
    std::cout << "Prob file path (enum): " << probPath2 << std::endl;
    ```

    **▶ 输出：**
    ```
    Prob file path: /tmp/hft/prob/binance/ethusdt_buy.csv
    Prob file path (enum): /tmp/hft/prob/binance/ethusdt_buy.csv
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 字符串版本
    prob_path1 = hft.PathManager.getProbFilePath("binance", "ethusdt", "buy")
    print(f"Prob file path: {prob_path1}")

    # 枚举版本
    prob_path2 = hft.PathManager.getProbFilePath(hft.Exchange.BINANCE, hft.CurrencyPair.ETH_USDT, hft.Side.BUY)
    print(f"Prob file path (enum): {prob_path2}")
    ```

    **▶ 输出：**
    ```
    Prob file path: /tmp/hft/prob/binance/ethusdt_buy.csv
    Prob file path (enum): /tmp/hft/prob/binance/ethusdt_buy.csv
    ```

### stat file

* 统计文件

```
/tmp/hft/stat/exchange/type/filename
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="4"
    #include "hft/util/path_manager.h"
    using namespace hft;

    std::string statPath = PathManager::getStatFilePath("binance", "trade", "btcusdt_20240101.csv");
    std::cout << "Stat file path: " << statPath << std::endl;
    ```

    **▶ 输出：**
    ```
    Stat file path: /tmp/hft/stat/binance/trade/btcusdt_20240101.csv
    ```

=== "Python"

    ```python hl_lines="3"
    import hft

    stat_path = hft.PathManager.getStatFilePath("binance", "trade", "btcusdt_20240101.csv")
    print(f"Stat file path: {stat_path}")
    ```

    **▶ 输出：**
    ```
    Stat file path: /tmp/hft/stat/binance/trade/btcusdt_20240101.csv
    ```

### data file

* 交易数据文件

```
/tmp/hft/data/exchange/channel/filename
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 枚举版本
    std::string dataPath1 = PathManager::getDataFilePath(Exchange::BINANCE, Channel::TRADE, "btcusdt_20240101.csv");
    std::cout << "Data file path (enum): " << dataPath1 << std::endl;

    // 字符串版本
    std::string dataPath2 = PathManager::getDataFilePath("okx", "ticker", "btcusdt_20240101.csv");
    std::cout << "Data file path: " << dataPath2 << std::endl;
    ```

    **▶ 输出：**
    ```
    Data file path (enum): /tmp/hft/data/binance/trade/btcusdt_20240101.csv
    Data file path: /tmp/hft/data/okx/ticker/btcusdt_20240101.csv
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 枚举版本
    data_path1 = hft.PathManager.getDataFilePath(hft.Exchange.BINANCE, hft.Channel.TRADE, "btcusdt_20240101.csv")
    print(f"Data file path (enum): {data_path1}")

    # 字符串版本
    data_path2 = hft.PathManager.getDataFilePath("okx", "ticker", "btcusdt_20240101.csv")
    print(f"Data file path: {data_path2}")
    ```

    **▶ 输出：**
    ```
    Data file path (enum): /tmp/hft/data/binance/trade/btcusdt_20240101.csv
    Data file path: /tmp/hft/data/okx/ticker/btcusdt_20240101.csv
    ```

### ipc command file

* zmq ipc command pub/sub file
* 用于**命令的发送**，命令的发送方可以将命令发送到这个地址；命令的执行方订阅这个地址负责执行
* 因为我们用pub/sub的方式来执行命令，**这个地址是命令发送方的标识**

```
/tmp/hft/zmq/command/name.ipc
```


!!! note "注意"

    * ipc command file和ipc file的用途不同
    * ipc command file用于**命令的发送**，例如：下单指令，撤单指令等等
        * `zmq/command/mm.ipc`，gateway进程可以订阅`mm.ipc`，就可以执行mm发送过来的下单指令
    * ipc file用于**消息的发送**，例如：ticker，trade的发送
        * `zmq/binance.trade.ipc`，mm进程可以订阅`binance.trade.ipc`，就可以收到binance的trade数据

**代码示例：**

=== "C++"

    ```cpp hl_lines="4"
    #include "hft/util/path_manager.h"
    using namespace hft;

    std::string zmqCmdPath = PathManager::getZmqIpcCommandFilePath("kraken_gateway");
    std::cout << "ZMQ command file path: " << zmqCmdPath << std::endl;
    ```

    **▶ 输出：**
    ```
    ZMQ command file path: /tmp/hft/zmq/command/kraken_gateway.ipc
    ```

=== "Python"

    ```python hl_lines="3"
    import hft

    zmq_cmd_path = hft.PathManager.getZmqIpcCommandFilePath("kraken_gateway")
    print(f"ZMQ command file path: {zmq_cmd_path}")
    ```

    **▶ 输出：**
    ```
    ZMQ command file path: /tmp/hft/zmq/command/kraken_gateway.ipc
    ```

### ipc file

* zmq ipc pub/sub file
* 用于**消息的发送**，消息的发送方可以将消息发送到这个地址，消息的消费方订阅这个地址消费消息
* 对于上层应用来说，从内部的ipc channel订阅消息和从交易所直接订阅消息在行为是上一致的。上层应用并不需要关心消息是从交易所直接推过来的，还是ipc channel推过来的

```
/tmp/hft/zmq/exchange.channel.ipc
```

**代码示例：**

=== "C++"

    ```cpp hl_lines="5 9"
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 枚举版本
    std::string zmqPath1 = PathManager::getZmqIpcFilePath(Exchange::BINANCE, Channel::TRADE);
    std::cout << "ZMQ IPC path (enum): " << zmqPath1 << std::endl;

    // 字符串版本
    std::string zmqPath2 = PathManager::getZmqIpcFilePath("okx", "ticker");
    std::cout << "ZMQ IPC path: " << zmqPath2 << std::endl;
    ```

    **▶ 输出：**
    ```
    ZMQ IPC path (enum): /tmp/hft/zmq/binance.trade.ipc
    ZMQ IPC path: /tmp/hft/zmq/okx.ticker.ipc
    ```

=== "Python"

    ```python hl_lines="4 8"
    import hft

    # 枚举版本
    zmq_path1 = hft.PathManager.getZmqIpcFilePath(hft.Exchange.BINANCE, hft.Channel.TRADE)
    print(f"ZMQ IPC path (enum): {zmq_path1}")

    # 字符串版本
    zmq_path2 = hft.PathManager.getZmqIpcFilePath("okx", "ticker")
    print(f"ZMQ IPC path: {zmq_path2}")
    ```

    **▶ 输出：**
    ```
    ZMQ IPC path (enum): /tmp/hft/zmq/binance.trade.ipc
    ZMQ IPC path: /tmp/hft/zmq/okx.ticker.ipc
    ```

### 🔥 shm file

* share memory file
* 用于跨进程通讯，可以共享市场的原始数据，也可以共享内部生成的信号，例如：Fair Price, Premium, Volatility等等

```
/tmp/hft/shm/exchange.type.symbol.valueType
```

详细映射规则参考：[共享内存路径](shm_path.md)

**代码示例：**

=== "C++"

    ```cpp
    #include "hft/util/path_manager.h"
    using namespace hft;

    // 方式1: Exchange, DataType, Currency, valueType
    std::string shmPath1 = PathManager::getShmFilePath(
        Exchange::BINANCE, 
        DataType::FAIR_PRICE, 
        Currency::BTC, 
        "single_quote"
    );
    std::cout << "Shm file path (Currency): " << shmPath1 << std::endl;

    // 方式2: Exchange, DataType, CurrencyPair, valueType
    std::string shmPath2 = PathManager::getShmFilePath(
        Exchange::BINANCE, 
        DataType::VOLATILITY, 
        CurrencyPair::BTC_USDT, 
        "pair_quote"
    );
    std::cout << "Shm file path (CurrencyPair): " << shmPath2 << std::endl;

    // 方式3: Exchange, Channel, CurrencyPair
    std::string shmPath3 = PathManager::getShmFilePath(
        Exchange::OKX, 
        Channel::TRADE, 
        CurrencyPair::ETH_USDT
    );
    std::cout << "Shm file path (Channel): " << shmPath3 << std::endl;

    // 方式4: 字符串版本 - 3参数
    std::string shmPath4 = PathManager::getShmFilePath("binance", "ticker", "btcusdt");
    std::cout << "Shm file path (3 params): " << shmPath4 << std::endl;

    // 方式5: 字符串版本 - 4参数
    std::string shmPath5 = PathManager::getShmFilePath("okx", "fair_price", "eth", "single_quote");
    std::cout << "Shm file path (4 params): " << shmPath5 << std::endl;
    ```

    **▶ 输出：**
    ```
    Shm file path (Currency): /tmp/hft/shm/binance.fair_price.btc.single_quote
    Shm file path (CurrencyPair): /tmp/hft/shm/binance.volatility.btcusdt.pair_quote
    Shm file path (Channel): /tmp/hft/shm/okx.trade.ethusdt
    Shm file path (3 params): /tmp/hft/shm/binance.ticker.btcusdt
    Shm file path (4 params): /tmp/hft/shm/okx.fair_price.eth.single_quote
    ```

=== "Python"

    ```python
    import hft

    # 方式1: Exchange, DataType, Currency, valueType
    shm_path1 = hft.PathManager.getShmFilePath(
        hft.Exchange.BINANCE,
        hft.DataType.FAIR_PRICE,
        hft.Currency.BTC,
        "single_quote"
    )
    print(f"Shm file path (Currency): {shm_path1}")

    # 方式2: Exchange, DataType, CurrencyPair, valueType
    shm_path2 = hft.PathManager.getShmFilePath(
        hft.Exchange.BINANCE,
        hft.DataType.VOLATILITY,
        hft.CurrencyPair.BTC_USDT,
        "pair_quote"
    )
    print(f"Shm file path (CurrencyPair): {shm_path2}")

    # 方式3: Exchange, Channel, CurrencyPair
    shm_path3 = hft.PathManager.getShmFilePath(
        hft.Exchange.OKX,
        hft.Channel.TRADE,
        hft.CurrencyPair.ETH_USDT
    )
    print(f"Shm file path (Channel): {shm_path3}")

    # 方式4: 字符串版本 - 3参数
    shm_path4 = hft.PathManager.getShmFilePath("binance", "ticker", "btcusdt")
    print(f"Shm file path (3 params): {shm_path4}")

    # 方式5: 字符串版本 - 4参数
    shm_path5 = hft.PathManager.getShmFilePath("okx", "fair_price", "eth", "single_quote")
    print(f"Shm file path (4 params): {shm_path5}")
    ```

    **▶ 输出：**
    ```
    Shm file path (Currency): /tmp/hft/shm/binance.fair_price.btc.single_quote
    Shm file path (CurrencyPair): /tmp/hft/shm/binance.volatility.btcusdt.pair_quote
    Shm file path (Channel): /tmp/hft/shm/okx.trade.ethusdt
    Shm file path (3 params): /tmp/hft/shm/binance.ticker.btcusdt
    Shm file path (4 params): /tmp/hft/shm/okx.fair_price.eth.single_quote
    ```