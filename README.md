

## 部署

### 下载

```
git clone git@github.com:relaxdl/hft-wiki.git
```

### 安装依赖

* `mkdocs-material`包含了MkDocs和Material主题
* 不需要单独装MkDocs

```
pip3 install mkdocs-material
```

### 本地运行文档服务器

```
mkdocs serve -a 0.0.0.0:8000
```

### 发布到Github Page

```
cd hft-wiki

# 创建虚拟环境
python3 -m venv venv

# 激活虚拟环境
source venv/bin/activate

# 安装 mkdocs & material主题
pip3 install mkdocs-material

mkdocs gh-deploy --force
```

## 如何新增文档

* 所有的文档以markdown的格式放在项目的docs目录下
* 文档在磁盘的目录结构和实际显示的目录结构不需要一致. 后期如果需要, 可以重新调整文件目录的展示组织形式, 不需要修改文件在磁盘的具体位置, 也不影响文档内置的文档站内超链接
* 文档默认是不会显示的, 要新增文档, 需要添加到`mkdocs.yml`中的`nav`部分

