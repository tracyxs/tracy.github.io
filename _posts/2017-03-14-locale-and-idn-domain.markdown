---
layout: post
title: "排查 locale 和 IDN 域名的问题"
date: 2017-03-14 16:00
tags: Troubleshooting
categories: 公众号
display: true
comments: true
---


## 问题描述

一个 Nagios 插件，用于通过 `whois` 检查域名是否到期。

然后仅有的两个中文域名之前还是好好的，现在报警了，提示找不到这个域名的 whois 记录。

但是机器上直接执行脚本，没有任何毛病。


## 排查

很明显中文域名有问题，非 ascii 码就是毛病多，想了下前阵子还修复了代码中对中文域名各种编码问题。

首先定位代码，传入的域名参数是 str，问题出在 Nagios 调用插件时，`subprocess` 调用 `whois` 命令上。

继而 `strace` 跟踪正常执行 `whois <idn-domain>` 和通过 Nagios 调用插件，发现 `locale` 这块有问题，后者未读取 `/usr/lib64/locale/locale-archive` 数据信息，最终正常情况下 IDN 会做 Punycode 转码，但是 Nagios 下最终并没有转码，还是字节码导致最终显示 whois 记录找不到这个域名……

然后在插件代码中打印 `locale` 值，本地默认设置的是 `zh_CN.utf-8` （不知道为啥设置为这个了），然后 Nagios 插件里是 `C`，扫了下 Nagios 代码，并没有 setlocale 的操作，比较好奇。

当然，这里 `locale` 中最终影响无法转码域名的是 `LC_CTYPE` 的配置。

测试可以用 `idn` 命令查询中文域名（idn命令是 `net-dns/libidn` 的一个接口工具），如果 `LC_CTYPE` 字符集设置正确，idn 会根据指定编码对 idn域名做 Punycode 转码为 ascii 域名：

```bash
$ LC_CTYPE=C idn 测试.com
idn: could not convert from ANSI_X3.4-1968 to UTF-8

$ LC_CTYPE=en_US.utf-8 idn 测试.com
xn--0zwm56d.com
```

最后尝试重启 Nagios，插件 locale 恢复为 `zh_CN.utf-8` 了。

后来经 @clanzx 提醒，想到以前遇到的一个问题，本地终端模拟器 iTerm2 设置了字符集，这个是可以配置开启转发到登录的远程机器上。因为**重启会根据当前 shell 的 locale 影响启动进程**，猜测可能有同事本地 locale 是 C，然后远程登录监控机器并转发了 locale 字符集，然后重启了Nagios。

然后就是解决代码问题了，**这个应该从代码根源保证 locale 一直是预期的**，而之前并没有这样的保证，其实一般情况下，还真没怎么去考虑这块……

尝试 Python 的 `locale.setlocale`，但是对 subprocess 中调用的命令不生效，哪怕 `shell=False`，奇怪。

在 subprocess 中调用命令时指定 `LC_CTYPE=en_US.utf-8 cmd`，但这样只能 `shell=True`。

最后尝试 `os.putenv()` 直接修改 locale 环境变量，这样就不依赖于当前 shell 的 locale 了，完美解决。

```python
os.putenv('LC_CTYPE', 'en_US.utf-8')
```

---

补充：

再看了下文档：

> os.putenv(varname, value)

> When putenv() is supported, assignments to items in os.environ are automatically translated into corresponding calls to putenv(); however, calls to putenv() don’t update os.environ, so it is actually preferable to assign to items of os.environ.

建议还是直接修改 `os.environ`：

```python
os.environ['LC_CTYPE'] = 'en_US.utf-8'
```
