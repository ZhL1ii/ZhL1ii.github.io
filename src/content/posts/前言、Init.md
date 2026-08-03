---
title: "MyGit(1): 前言与 git init"
pubDatetime: 2026-08-04 00:04:00+08:00
description: "自底向上实现一个 mini-Git"
---

# MyGit(1): 前言与 git init

这套文章旨在自底向上实现一个 Git。Git 是一个庞大的项目，功能繁多，但其核心是很简单的，这很反直觉。但是跟着博文实现出一个与 Git 功能近乎完全兼容的 mini-Git，你将彻底理解 Git 核心。

在开始之前，你需要掌握一些基本的 Git 操作 (init、add、commit、checkout)、Python 的基本语法以及一些基本的 Shell。

> 博文中的代码均可写入同一个 Python 文件中。
>
> 由于本次实现的是一个 **Unix** 风格的终端程序，所以 Windows 中尽量使用 WSL 。
>
> 请保证 Python 版本在 3.10 及以上。

在你的项目目录下创建两个文件：

- 一个可执行文件，名为 `mygit`
- 一个 Python 库，名为`libmygit.py`

mygit 中写入：

```python
#!/usr/bin/env python3

import libmygit
libmygit.main()
```

在终端中执行，使其变为可执行文件：

```sh
chmod +x mygit
```

接下来要在 `libmygit.py` 中导入要用到的所有包。

- Git 是一个 CLI 程序，显然需要一个解析命令行参数的库，这里使用 [argparse](https://docs.python.org/3/library/argparse.html)

```py
import argparse
```

- Git 使用 INI 格式的配置文件，configparser 库可以读写这些文件

```python
import configparser
```

- 后续需要进行日期、时间相关的操作

```python
from datetime import datetime
```

- 后续需要读取一次用户（组）数据。

```python
try:
    import grp, pwd
except ModuleNotFoundError:
    pass # 这个包在 Windows 上不可用
```

- 为了支持 `.gitignore` ，我们需要匹配文件名与后缀。

```python
from fnmatch import fnmatch
```

- Git 经常使用哈希函数，所以需要用到 hashlib 库。

```python
import hashlib
```

- 在实现暂存区 / index 时，会用到 ceil 函数。

```python
from math import ceil
```

- 操作文件系统

```python
import os
```

- 正则表达式

```python
import re
```

- 访问命令行参数

```python
import sys
```

- 使用 zlib 压缩内容

```python
import zlib
```

至此导包完成。
