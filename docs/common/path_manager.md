# 路径管理

## 介绍

* hft-lib库提供了路径管理的功能，项目中用到的所有路径。由底层C++的`PathManager`来统一管理，方便维护
* 跨进程和跨语言的访问，只要hft root设置的一致，就能保证文件路径的唯一性。例如：我们需要用文件的inode信息生成跨进程共享的share memory key的时候，这个模块可以保证唯一性
* 这个模块只负责返回路径，并不会创建路径和文件，其它模块负责文件和路径的创建
* 提供了C++和Python两套API给上层应用使用

## 具体实现

C++实现 🔒 *（私有仓库，需要授权访问）*

* [path_manager.h](https://github.com/relaxdl/hft-lib/blob/main/include/hft/util/path_manager.h)
* [path_manager.cpp](https://github.com/relaxdl/hft-lib/blob/main/src/hft/util/path_manager.cpp)

## 定义

### hft root

* 是一个目录，后续提到的所有路径都在这个根目录下；系统内所有的文件都保存在一个根目录下，按照不同的用途分类组织
* 如果不设置，系统默认的hft root是：`/tmp/hft`

**代码示例：**

=== "C++"

    ```cpp
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

    ```python
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

    ```cpp
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

    ```python
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

    ```cpp
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

    ```python
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

    ```cpp
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

    ```python
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

    ```cpp
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

    ```python
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

```
/tmp/hft/stat/exchange/type/filename
```

### data file

```
/tmp/hft/data/exchange/channel/filename
```

### zmq ipc command file

```
/tmp/hft/zmq/command/name.ipc
```

### shm file

```
/tmp/hft/shm/exchange.type.symbol.valueType
```