# Sphinx + github + readthedocs 托管文档

## 一、Sphinx配置 

### 1. 安装Sphinx、及其插件

```
pip3 install sphinx sphinx_rtd_theme recommonmark sphinx-markdown-tables sphinxemoji
```

### 2. 初始化:

```bash
# 创建文档根目录
mkdir -p my-cookbook
cd my-cookbook
# 可以回车按默认配置来写
sphinx-quickstart
```

下面是我填写的，其他基本上默认即可：

```
> Separate source and build directories (y/n) [n]:y
> Project name: my-cookbook
> Author name(s): wu
> Project version []: 0.1
> Project release [1.0]: 0.1
> Project language [en]: zh_CN
```

然后运行 `tree -C .` 查看生成的sphinx结构:

```
.
├── build
├── make.bat
├── Makefile
└── source
    ├── conf.py
    ├── index.rst
    ├── _static
    └── _templates
```

添加一篇文章，在source目录下新建hello.rst，内容如下:

```
hello,world
=============
```

`index.rst` 修改如下:

```
.. toctree::
   :maxdepth: 2
   :caption: Contents:

   hello
```

预览效果

然后在更目录执行`make html`，进入`build/html`目录后用浏览器打开`index.html`

### 3. 配置Sphinx主题，插件

#### 3.1 配置主题

修改`./source/conf.py`配置文件，添加配置如下：

```python
import sphinx_rtd_theme
html_theme = "sphinx_rtd_theme"
html_theme_path = [sphinx_rtd_theme.get_html_theme_path()]

# The master toctree document.
master_doc = 'index'
```

#### 3.2 拓展插件支持

| 插件                   | 作用              |
| ---------------------- | ----------------- |
| recommonmark           | markdown          |
| sphinx-markdown-tables | markdown 语法表格 |
| sphinxemoji            | 颜文字支持        |

添加在根目录下添加环境依赖文件，即`./requirements.txt` pip要求文件（**Readthedocs配置**时需要用到）

```bash
sphinx-autobuild==2020.9.1
sphinx-markdown-tables==0.0.15
sphinx-rtd-theme==0.5.0
sphinxcontrib-applehelp==1.0.2
sphinxcontrib-devhelp==1.0.2
sphinxcontrib-htmlhelp==1.0.3
sphinxcontrib-jsmath==1.0.1
sphinxcontrib-qthelp==1.0.3
sphinxcontrib-serializinghtml==1.1.4
```

预览效果同上，执行 `make html` 后，进入`build/html`目录后用浏览器打开`index.html`

修改`./source/conf.py`配置文件，添加配置如下：

```python
from recommonmark.parser import CommonMarkParser

extensions = [
    "recommonmark",
    "sphinx_markdown_tables",
    # 'sphinxemoji.sphinxemoji',
]

# Add any paths that contain custom static files (such as style sheets) here,
# relative to this directory. They are copied after the builtin static files,
# so a file named "default.css" will overwrite the builtin "default.css".
html_static_path = ["_static"]

source_parsers = {
    ".md": CommonMarkParser,
}
source_suffix = [".rst", ".md"]
```

### 4. 自动编译

每次改完，如果都生成新的 `html` 文件，都需要执行 `make html`。那我们可以通过 git 的 `.git/hooks/pre-commit` 都执行，`git commit`之前都会执行这个脚本。这样就达到了，每次改完代码，我们 `git commit` 之前，都先自动编译，然后再提交即可

vim .git/hooks/pre-commit

```bash
#!/bin/sh

make html
git add -A
```

然后赋值权限

```bash
chmod +x .git/hooks/pre-commit
```

这样每次改完后，`git add -A`  -> `git commit -am 'xxx'` -> `git push origin master` 即可

## 二、托管 readthedocs.org

[Read the Docs](https://readthedocs.org/)是一个开源项目文档托管和阅读工具。它提供了Sphinx文档的多种阅读方式，它主要有以下特点：

- 支持多种形式的阅读，`web/pdf/epub`等，同时可以全文搜索
- 支持文档的版本控制，`git/svn`等
- 支持对`github/gitlab`等仓库中某个标签或分支托管的Sphinx文档的clone、build
- 支持[webhooks](https://www.cnblogs.com/wangwangever/p/7467142.html)，当版本控制下的文档有更新时就会自动触发build文档

这里大概讲下工作原理：

- 进入[Read the Docs官网](https://readthedocs.org/)点击登录，输入用户、邮箱、密码完成注册
- 绑定 github 账号 - 导入 github 仓库作为项目 - 为项目创建名称、指定代码库地址
- 通过 webhooks 实现当版本控制，当前版本库有更新时就会自动触发 build 文档，即我们 `git push origin master` 这样推上远程分支后，readthedocs 就会自动编译，时间大概是几分钟不等。

具体配置参考文档 [ReadtheDocs](https://docgenerate.readthedocs.io/en/latest/sphinx/3-release/1-docs.html) 即可

## 三、常见问题

#### 1. `readthedocs.org` 编译找不到相应的库 
如果网站托管在 `readthedocs.org` 上，看到 build 时出现异常，例如 `ModuleNotFoundError: No module named 'sphinx_markdown_tables' `。

   这是因为在编译时，没有这个环境，应该就是没有上传 `requirements.txt` 文件

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/sphinx/sphinx_markdown_tables.png)


解决方式是添加环境依赖文件，在顶级目录下创建即可，vim requirements.txt`

```bash
sphinx-markdown-tables==0.0.15
```

`git push origin master` 推上去之后，就可以看到是正常构建完成了。里面有一条执行语句是 `python -m pip install --exists-action=w --no-cache-dir -r requirements.txt` ，点开看这条语句，就发现它会安装好 `requirements.txt` 里的环境变量了。

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/sphinx/requirements.png)