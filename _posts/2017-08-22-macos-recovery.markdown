---
layout: post
title: "MacOS 启动失败和恢复记录"
date: 2017-08-22 22:00
categories: 公众号
display: true
comments: true
---

![](https://tankywoo-wb.b0.upaiyun.com/gzh/20170822-macos-recovery.jpeg)

昨天很倒霉的遇到 Mac 升级的故障了。

早上出发前想着笔记本已经很久没关机了，就关了休息会，到公司后发现开机出问题了，加载条加载到一半屏幕就黑了，然后屏幕间歇性的闪一下，并且画面是混乱的。

重启试了几次都不行，于是网上搜了下，先通过 `Command + R` 进入『恢复模式』，发现屏幕显示正常，首先排除屏幕问题。

看到有磁盘工具，检查磁盘，发现磁盘没有故障。于是进去看了下，不过这种方式只能看到文件，但是内容看不了。

初步怀疑是启动loader有问题导致，但是还不能确认。

继续看网上方案，通过 `Command + Option + P + R` 重置 NVRAM，依然不行。

然后尝试 `Command + S` 进去『单用户』模式，这个是解决重置密码用的。其实和很多 Linux 发行版一样，提供一个救援恢复的功能。里面也可以看到文件都正常。

正好在上周二时，隔了3个月之久作了一次 TimeMachine 备份，再上次就是5月份的备份了。想着平时的一些事项主要都是远程登录到其它机器上弄，Mac下用得少，把一些能想到的数据备份出来就行了，其余就恢复到上周的备份。

但是在单用户模式下挂载U盘备份遇到了问题，首先将U盘格式化为 `exfat`，无法挂载，虽然有 `mount_exfat` 程序，然后想着和苹果的文件系统保持一致，用 `hfs/hfs+`，依然无法挂载，最终发现 `msdos` 挂载正常：

```
mount_msdos /dev/rdisk1s1 /Volume/udisk
```

然后就是将一些这几天产生的数据备份出来。接着再次重启进入恢复模式，从 TimeMachine 开始恢复到上周的备份。花了两个小时终于恢复备份，自动重启依然和之前情况一样。想了想感觉就是之前怀疑的问题，因为我的 MAS 是开启了自动更新，并且电脑很久没有重启，所以有可能在上周这个备份之前启动器就已经有问题了，只是没有重启所以不知道。

再次尝试恢复为三个月前的备份，又等了两个小时，重启时看着进度条以比以前慢了不少的节奏缓慢加载，心里还是有些紧张，还好和我预测的一样，是启动器的问题，这个老的备份可以正常工作。并且在恢复模式也可以看到，最近这个备份和上一次备份的系统小版本是不一致的。

接着就是恢复数据了，幸好上周给备份了下，挂载上 TimeMachine，直接找到挂载卷里相应的备份目录，开始逐个目录检查有哪些变更过的，然后同步数据。

在这个过程中，临时写了几个脚本用于排查。

这个脚本是用于获取当前目录下子目录的 md5sum 或者指定目录的 md5sum，针对小目录比较合适，大目录会比较慢。

```bash
#!/bin/bash

if [ -n "$1" ]; then
    echo -e -n "$1\t"
    find -s "$1" -type f -exec md5sum {} \; | md5sum
else
    for d in "$PWD"/*; do
        subdir=$(basename "$d")
        echo -e -n "$subdir\t"
        find -s "$subdir" -type f -exec md5sum {} \; | md5sum
    done
fi
```

然后就是 `rsync` 对比并同步数据或者 `cp` 也可以：

```
$ rsync -hvarH --stats --progress <path-to-timemachine> <path-to-local>
```

在 `cp` 时遇到一些问题，发现复制过来的文件夹没有写权限：

```
$ cp -a <src> <dst>
$ ls -al
drwxr-xr-x@ 32 TankyWoo  staff   1.1K Jul  3 10:41 mydir
```

注意权限那里的 `@` 表示有扩展属性和ACL：

```
# -@ 看扩展属性
$ ls -@l
drwxr-xr-x@ 32 TankyWoo  staff   1.1K Jul  3 10:41 mydir
        com.apple.metadata:_kTimeMachineNewestSnapshot    50B
        com.apple.metadata:_kTimeMachineOldestSnapshot    50B

# -e 看ACL
drwxr-xr-x@ 32 TankyWoo  staff   1.1K Jul  3 10:41 mydir
 0: group:everyone deny add_file,delete,add_subdirectory,delete_child,writeattr,writeextattr,chown
```

因为复制文件时习惯性的保留所有权限，所以通过 `cp -a` 拷贝的 TimeMachine 数据，`-a` 会保留一样的扩展属性和ACL，改为 `cp -r` 即可。或者通过上面给的 `rsync` 命令，貌似 Mac 下 rsync 也没有支持保留扩展属性的参数。

清除目录或文件的ACL：

```
chmod [-R] -N <dir_or_file>
```

另外，如 `ls -@l` 看到的这些属性，和 TimeMachine 相关，比较碍眼，可以通过 `xattr -d` 删除：

```
xattr [-r] -d com.apple.metadata:_kTimeMachineNewestSnapshot <dir_or_file>
```

花了两天时间，终于恢复的差不多了，包括一些本地笔记软件或如 1p 这些服务的存数数据。

总结下这次的教训吧：

- 关闭 MAS 自动更新，并且在手动更新前，一定要先做一次备份，血的教训！
- TimeMachine 一定要用起来，而且最好保持2、3天内就备份一次
- 虽然 Mac 基本不用关机，但是系统层面软件更新后还是先重启吧。还好阴错阳差的最后两次备份间隔较久，否则一直没重启而备份的话估计正常备份都被覆盖了
- 本地一些软件的数据和文档，能用网盘如 Dropbox 存储的就存储起来吧，或者用 Git 等版本管理工具，并及时提交
