---
layout: post
title: "定位 rsyslog v8 的主机名是 [localhost] 问题"
date: 2017-06-24 14:00
comments: true
---

（本来这篇也准备用英文写的，想想还是算了，中文吧，中英全凭心情，虽然自己的英语弱爆了……）

rsyslog 是我到现在为止，troubleshooting 次数算相当多的一个工具了，没办法，文档乱的辣眼睛就不说，一路过来几个版本的配置语法变更太多，还有以前遇到的 RELP 传输阻塞等问题，心累。

昨天下午排查这个问题把脑袋都搞疼了。


## 问题描述

Ubuntu 12.04 下，官方源稳定的 rsyslog 版本是 5.8.6。现在 rsyslog 已经逐步稳定到 v8 版本了。所以前一阵子将一些系统都通过 PPA 源升级到 v8 版本了。

不过后来发现**个别机器**重启后有问题，日志格式是 `RSYSLOG_TraditionalFileFormat`，大概就是这样：

```
$template TraditionalFileFormat,"%TIMESTAMP% %HOSTNAME% %syslogtag%%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n"
```

但是 `%HOSTNAME%` 字段的值并不是主机名，而是 `[localhost]`。

几个关键字：

* 个别机器，并不是所有机器
* 只在启动时，如果在启动后重启 rsyslog 进程，那么主机名又正常了
* 出问题时 `$HOSTNAME%` 的值是 `[localhost]`

关于为何个别机器启动时 rsyslog 使用 `[localhost]`，但是重启进程主机名又正常，这一点我还没有排查出原因，感觉比较奇怪，而且这个问题不好复现，偶尔又正常，可能和 `upstart` 配合有问题，后者没有深究过。

目前主要定位在普遍情况下 v8 版本使用 `[localhost]` 的原因。

## 定位问题

在 v5 版本下，我重来没见过这个问题，但是在 v8 下，个别机器出现了。

于是我从 Github 上拉了 rsyslog 的源码，先分析了下 v8 的情况。

排查时将问题机器同步一份到虚拟机，并且随便设置了一个主机名，通过 `rsyslog -dn` 在前台运行并开启 debug，看到几条关键信息：

```
# 信息1
getaddrinfo: Name or service not known

# 信息2
GenerateLocalHostName uses '[localhost]'
```

而在 v5 版本，只会看到：

```
GenerateLocalHostName uses '<预期的主机名>'
```

很明显，在 v8 版本，作了 DNS 解析并且无法找到主机名，因为本地虚拟机上，这个主机名并没有设置 DNS 记录。

定位 `runtime/glbl.c`：

```c
static rsRetVal
GenerateLocalHostNameProperty(void)
{
    ...
    prop_t *hostnameNew;
    uchar *pszName;
    ...

    if(LocalHostNameOverride == NULL) {
        if(LocalHostName == NULL)
            pszName = (uchar*) "[localhost]";
        else {
            if(GetPreserveFQDN() == 1)
                pszName = LocalFQDNName;
            else
                pszName = LocalHostName;
        }
    } else { /* local hostname is overriden via config */
        pszName = LocalHostNameOverride;
    }
    DBGPRINTF("GenerateLocalHostName uses '%s'\n", pszName); 
    ...
```

如果 rsyslog 没有强制配置主机名，则确认 rsyslog 是否配置了 `PreserveFQDN`，若没有，则 `%HOSTNAME%` 使用本地的主机名。v5 版本是使用主机名，这个是预期的。但是这块并不能看出 `LocalFQDNName` 和 `LocalHostName` 是如何获取的。

进一步，在 `tools/rsyslogd.c` 主代码中，在 `GenerateLocalHostNameProperty()` 之前调用了 `queryLocalHostname()`，而 `queryLocalHostname()` 根据 `runtime/net.c` 的 `getLocalHostname()` 获取 FQDN，然后根据 FQDN 设置 `LocalFQDNName` 和 `LocalHostName`。在 `getLocalHostname()` 中核心逻辑就是 `gethostname()` 获取主机名，然后 `getaddrinfo()` 作 DNS 解析查询 IP，因为域名没有设置，所以无法解析导致如上的 `getaddrinfo: Name or service not known` 报错，进而设置 `[localhost]` 作为主机名了。

接着看 v5 版本的实现，`GenerateLocalHostNameProperty()` 的实现基本一样，而在 `tools/rsyslogd.c` 主代码中直接调用 `runtime/net.c` 的 `getLocalHostname()`，`getLocalHostname` 的实现逻辑仅仅是简单的 `gethostname()`，在定义 `%HOSTNAME%` 这块并没有去解析。

在 DNS 机器上实际抓包也可以看到，机器启动时启动 rsyslog 进程，并作了主机名的 A 记录查询。

目前排查有两个方法：

一是通过 rsyslog 配置 `$LocalHostName` 为主机名即可，这样导致 `LocalHostNameOverride` 不为 `NULL` 然后主机名直接设置为它。二是修改 `/etc/hosts` 增加主机名的记录。推荐使用前者。

这块不知道作者为何非要去给主机名做一个解析，还有就是理顺了这块的逻辑，还是不清楚最开始的问题，为何只有个别机器在启动时没有获取主机，我将 `/etc/init/rsyslog.conf` 启动加上了 `-d` 开启 DEBUG 并设置 `RSYSLOG_DEBUGLOG` 环境变量指定 DEBUG 日志，看到也是 `getaddrinfo: Name or service not known`，但是实际上在 DNS 服务器上抓包看到有 DNS 解析并返回记录了。

目前只能先设置 `$LocalHostName` 来解决了。

---

2017-06-26 补充：

因为想起来 Gentoo 下 rsyslog 的版本还是挺新的，然后看了下，稳定版是 `v8.26.0` 的版本（而 Ubuntu PPA 源最新版是 `v8.27.0` 版本），看了下这个版本的代码，是没有引入之前 `getaddrinfo` 的问题，这个版本使用 `gethostbyname()` 来解析，如果失败则直接将 FQDN 置为 `gethostname()` 获取的主机名；在 `v8.27.0` 下如果 `getaddrinfo()` 失败后则直接退出不再进行后面的操作。

然后我又将 Gentoo 下的 rsyslog 升级到 `v8.27.0` 这个非稳定版，奇怪的是主机名正常！最后排查 ebuild 文件发现 Gentoo 官方额外有一个 patch 专门修复这个问题：

```diff
$ cat /usr/portage/app-admin/rsyslog/files/8-stable/rsyslog-8.27.0-fix-hostname-detection-when-getaddrinfo-fails.patch
From 1a7d3a088969b47798bc1da712ca2772f91a7c02 Mon Sep 17 00:00:00 2001
From: Jiri Vymazal <jvymazal@redhat.com>
Date: Wed, 31 May 2017 16:26:56 +0200
Subject: [PATCH] Ignoring NONAME error from getaddrinfo so we have hostname
 set even without working network

---
 runtime/net.c | 6 +++++-
 1 file changed, 5 insertions(+), 1 deletion(-)

diff --git a/runtime/net.c b/runtime/net.c
index 2d8de9429..edffc677a 100644
--- a/runtime/net.c
+++ b/runtime/net.c
@@ -1188,7 +1188,11 @@ getLocalHostname(uchar **ppName)
                memset(&flags, 0, sizeof(flags));
                flags.ai_flags = AI_CANONNAME;
                int error = getaddrinfo((char*)hnbuf, NULL, &flags, &res);
-               if (error != 0) {
+               if (error != 0 &&
+                   error != EAI_NONAME && error != EAI_AGAIN && error != EAI_FAIL) {
+                       /* If we get one of errors above, network is probably
+                        * not working yet, so we fall back to local hostname below
+                        */
                        dbgprintf("getaddrinfo: %s\n", gai_strerror(error));
                        ABORT_FINALIZE(RS_RET_IO_ERROR);
                }

```

赞！

然后我又瞄了下 Github rsyslog master 分支的原因，[目前这个问题也修复了](https://github.com/rsyslog/rsyslog/commit/1a7d3a088969b47798bc1da712ca2772f91a7c02)（当初直接看的 v8.27.0 分支，忽略了最新的代码）。

估计下一个版本就会包含修复这个问题。
