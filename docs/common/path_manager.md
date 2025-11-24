# 路径管理

## 介绍

* hft-lib库提供了路径管理的功能，项目中用到的所有路径。由底层C++的`PathManager`来统一管理，方便维护
* 跨进程和跨语言的访问，只要hft root设置的一致，就能保证文件路径的唯一性。例如：我们需要用文件的inode信息生成跨进程共享的share memory key的时候，这个模块可以保证唯一性
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

```
/tmp/hft/tmp/exchange/dataType/filename
```

### model file

```
/tmp/hft/model/exchange/filename
```

### return file

```
/tmp/hft/return/exchange/symbol.csv
```

### prob file

```
/tmp/hft/prob/exchange/symbol_side.csv
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