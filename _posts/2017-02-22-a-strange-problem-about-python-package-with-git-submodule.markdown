---
layout: post
title: "排查Python打包时发布文件列表对Git子模块的奇怪行为"
date: 2017-02-22 22:00
tags: Python,Troubleshooting
categories: 公众号
display: true
comments: true
---


## 问题描述

项目：simiki；使用 `git` 维护。

在做单元测试时，发现 `nosetest` 是没有问题，但是 `tox` 是会报错，具体发生在先打包然后 installpkg 步骤上，大致报错信息：

```
  Failed building wheel for simiki
  Running setup.py clean for simiki
Failed to build simiki
Installing collected packages: simiki
  Found existing installation: simiki 1.6.0.1
    Uninstalling simiki-1.6.0.1:
      Successfully uninstalled simiki-1.6.0.1
  Running setup.py install for simiki: started
    Running setup.py install for simiki: finished with status 'error'
    Complete output from command /.../simiki/.tox/py27/bin/python2.7 -u -c "import setuptools, tokenize;__file__='/tmp/pip-abt7sB-build/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code
, __file__, 'exec'))" install --record /tmp/pip-PAIDR2-record/install-record.txt --single-version-externally-managed --compile --install-headers /.../simiki/.tox/py27/include/site/python2.7/simiki:
    running install
    running build
    ...
    reading manifest file 'simiki.egg-info/SOURCES.txt'
    reading manifest template 'MANIFEST.in'
    writing manifest file 'simiki.egg-info/SOURCES.txt'
    creating build/lib/simiki/conf_templates
    copying simiki/conf_templates/Dockerfile -> build/lib/simiki/conf_templates
    copying simiki/conf_templates/_config.yml.in -> build/lib/simiki/conf_templates
    copying simiki/conf_templates/fabfile.py -> build/lib/simiki/conf_templates
    copying simiki/conf_templates/gettingstarted.md -> build/lib/simiki/conf_templates
    creating build/lib/simiki/themes
    error: can't copy 'simiki/themes/simple': doesn't exist or not a regular file  // 发生错误！！！
```

提示无法**复制** `simiki/themes/simple`，看输出是当做普通文件做复制，但是这个实际上是目录。

---

## 排查

（整个排查过程加起来花了估摸有3个小时，大部分是到处乱撞……）

首先确认了下，大致就是 `sdist` 生成 zip 包，然后 pip 安装 zip 包，但是在安装过程中，复制包中文件到相应位置时失败了。

解压 zip 包，发现里面的 `simiki.egg-info/SOURCES.txt` 中包含了 `simiki/themes/simple` 和 `simple/themes/simple2` 这两行，而对于同样是目录的 `simiki/conf_templates` 则并没有。`simiki.egg-info` 记录了打包的一些元信息，`SOURCES.txt` 里就是安装时复制的**文件**列表，正常是不会包含目录自身的，直接会列出目录中的文件。

最近给项目中的两个默认主题 `simple` 和 `simple2` 改为 `git submodule` 来维护了，具体见 commit [`4f10e16`](https://github.com/tankywoo/simiki/commit/4f10e165230dfec1bd5a70d37a2478fb3131a189)。

而出问题的正是子模块目录。且如果 `git checkout` 到变更为 submodule 之前的 commit 时，是没有这两行，即不会有问题。

而对于控制打包包含的文件，是 `MANIFEST.in` 来处理的。

然后就是尝试：

```bash
python setup.py egg_info; grep -i simple2 simiki.egg-info/SOURCES.txt
```

或者

```bash
python setup.py sdist
```

以及 `strace` 跟踪，不过信息太多，看不出所以然。

想到可能和 `git` 有关，因为 `strace` 时看到一堆尝试读 `.git` 下各个文件的内容。

因为是在配置了 `submodule` 之后才发生的，所以考虑把相关文件如 `.gitmodules`、`.git/modules`、`simiki/themes/simple/.git` 都删掉了，还是不行。

但是如果直接删除 `.git` 目录后，发现 `python setup.py egg_info` 里就没有主题目录名自身了。

继续测试，将 `.git/objects/*` 删除还有那两行，但是如果将 `.git/objects` 目录直接删除，则会报如下错误并且没有那两行了。

```
...
writing entry points to simiki.egg-info/entry_points.txt
writing manifest file 'simiki.egg-info/SOURCES.txt'
fatal: Not a git repository (or any parent up to mount point /home)
Stopping at filesystem boundary (GIT_DISCOVERY_ACROSS_FILESYSTEM not set).
reading manifest file 'simiki.egg-info/SOURCES.txt'
reading manifest template 'MANIFEST.in'
writing manifest file 'simiki.egg-info/SOURCES.txt'
...
```

基本可以确认在 `git` 这块的问题。

中途又看了下Python打包的相关文档 [Creating a Source Distribution](https://docs.python.org/2/distutils/sourcedist.html#manifest)

里面说明了打包包含文件的默认行为，并且提及会忽略各种版本管理库的元信息，如 `.git`。我就好奇那为何还会读取这个目录下所有问题？

继续。

如果删除 `.git`，`SOURCES.txt` 里要少很多文件，包括 `.coverage`，`.travis.yaml` 等，都是没有显式加在 `MANIFEST.in` 中的。

后来想到：**减少的文件都是没有显式添加，但是在 git 库中维护；并且，作为子模块，其在 git 中存储实际是一个特殊文件，记录源仓库位置**

**猜测：setuptools 会读取 git 树里的文件，将这些也加入到列表中，效果上如：`git ls-files` 所看到的。（可参考以前写的[底层命令例子](https://wiki.tankywoo.com/book/version-control-with-git.html#_2)）**

目前的解决办法就是对这两submodule主题的目录名排除：

```diff
diff --git a/MANIFEST.in b/MANIFEST.in
index 0524db0..626d4e0 100644
--- a/MANIFEST.in
+++ b/MANIFEST.in
@@ -4,3 +4,5 @@ include LICENSE
 include *requirements.txt
 include tox.ini
 recursive-include simiki *.html *.css *.in *.md *.py
+exclude simiki/themes/simple
+exclude simiki/themes/simple2
```

<strike>**TODO**：猜测最终没有实际代码的支持……</strike>

---

后来去吃了个饭，清醒下继续从源码处入手。

首先定位到是 `setuptools/command/egg_info.py` 中的 `add_defaults()` 方法里添加了额外的这些文件。

（这块被坑了下，看到里面调用了 `distutils` 模块中的 `findall()` 方法，而这个方法又调用了一个 `findall()` 函数，虽然 `distutils/filelist.py` 中有这个函数了，但是实际是调用 `setuptools/__init__.py` 中的同名函数，函数名被覆盖了。坑爹的习惯了打 `print` 来查看信息，结果怎么都没看出是怎么调用的，改 `pdb` 才发现不是用的这个函数。）

继续看这个方法：

```python
def add_defaults(self):
    sdist.add_defaults(self)
    self.filelist.append(self.template)
    self.filelist.append(self.manifest)
    rcfiles = list(walk_revctrl())
    if rcfiles:
        self.filelist.extend(rcfiles)
    elif os.path.exists(self.manifest):
        self.read_manifest()
    ...
```

其中，`sdist.add_defaults(self)` 会添加一些 sdist 默认的文件，也就是上面文档中提到的规则，包括 `setup.py`、`setup.cfg`、`simiki/*.py`、`tests/test_*.py`等。

接着，下面的 `walk_revctrl()` 通过 `entry_point` 来调用 `setuptools_scm/integration.py` 中的 `find_files()` 函数，这个函数目前支持 `hg` 和 `git` 的检查，查看项目目录下是否有 `.hg` 和 `.git` 目录，比如 simiki 这个使用 git 维护的，所以找到 `.git` 目录，最后调用 `setuptools_scm/git.py` 中的 `FILES_COMMAND` 变量中的命令，而这个命令就是我之前的猜想：

```python
# site-packages/setuptools_scm/git.py
FILES_COMMAND = 'git ls-files'
```

OK，这个问题算圆满解决了，目前来看，针对 `submodule`，只能在 `MANIFEST` 中将其手动排除，但是子模块初始化后里面的文件要 `recursive-include` 进来。

---

*2017-02-26 补充：*

基于上面的现象，发现在自己的 Mac 环境下和 Gentoo 环境下得到的 SOURCES.txt 结果不一样，在 Mac 下打包的元信息里还是只包括以前的那些文件，并没有从 `git ls-files` 中获取。

排查后发现是 `setuptools_scm` 这个包在 Mac开发机上没有装。

先前没多想，还以为 `setuptools_scm` 是 `setuptools` 的一个依赖包，是必装的……

Gentoo 下 `setuptools_scm` 被装上是因为 `bpython` 这个包通过 `emerge` 安装时，中间有多层依赖，最终有个包依赖 `setuptools_scm` 从而导致其装上。

因为我的 simiki 开发环境是通过 virtualenv 创建的，有个选项 `--no-site-packages`，以前 virtualenv 创建虚拟环境时，默认是包括系统的包，即 `--system-site-packages`， 后面 `--no-site-packages` 被标记弃用并置位默认选项，表示不链接系统上装的 python 包。

这样相当于我的开发虚拟环境是没有 `setuptools_scm` 这个包了。

刚去 Github 上看了一些知名的项目，没有看到有针对这个问题的处理，如 Tornado 也是手动指定一堆文件去 include，flask 也是类似。

个人考虑为了杜绝这种奇葩的现象：

1. 显式的 exclude 不需要打包进去的文件，如 `.coveragerc`，`.travis.yml` 等
2. 先打包不要上传，看看都有哪些文件，检查一下
