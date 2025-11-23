
## 下载

```
git clone git@github.com:relaxdl/hft-wiki.git
```

## 安装依赖

* `mkdocs-material`包含了MkDocs和Material主题
* 不需要单独装MkDocs

```
pip3 install mkdocs-material
```

## 本地运行文档服务器

```
mkdocs serve -a 0.0.0.0:8000
```

## 发布到Github Page

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