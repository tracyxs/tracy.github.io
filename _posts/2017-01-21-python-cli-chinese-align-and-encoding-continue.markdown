---
layout: post
title: "Python 终端下中文字符对齐处理和编码续"
date: 2017-01-21 21:00
categories: 公众号
display: true
comments: true
---

本来是修改自己一个终端小程序的宽字符处理，然后就和编码纠结上了。

以前总结过一篇 [Python Encoding]({{ site.baseurl }}{% post_url 2015-01-21-python-encoding %})。

这两天花了不少时间继续研究了下这块，越研究越让人迷糊，还存在不少疑问。只能说在研究和总结这块时，我的内心是崩溃的……希望以后不再在这块纠结。

以下总结在环境 Linux，Python2.7 下研究。

先谈谈终端下中文字符（宽字符）的对齐输出问题：


## 终端下中文字符（宽字符）的对齐输出问题

比如我在终端下输出表格，里面包含了中英文，因为中英文的长度不一致，`len()`获取的宽度是编码字节的长度，不是实际长度：

```python
>>> len(u'我'.encode('utf-8'))
3
>>> len(u'我'.encode('gbk'))
2
```

这里「我」如果是 utf-8 编码，则占 3 个字节长度，而在 gbk 下则是 2 个字节长度。所以通过`len()`来固定长度显然不合适，造成无法对齐的情况。

后来看到[这个帖子](http://stackoverflow.com/questions/23320148/how-to-control-output-format-when-chinese-characters-and-ascii-mixed/23320535#23320535)，了解到`unicodedata`这个库，因为最终的 unicode 码点是唯一的，所以通过这个库，可以获取某个 unicode 字符的宽度。

我的处理：

```python
#coding: utf-8
import unicodedata
def wide_chars(s):
    """return the extra width for wide characters
    ref: http://stackoverflow.com/a/23320535/1276501"""
    if isinstance(s, str):
        s = s.decode('utf-8')
    return sum(unicodedata.east_asian_width(x) in ('F', 'W') for x in s)

# example
max_width = 20
s1 = u'中文'
# Format Specification Mini-Language
# ref: https://docs.python.org/2/library/string.html#format-specification-mini-language
fmt_str1 = u'{0:<%s}|' % (max_width - wide_chars(s1))
s2 = u'ab'
fmt_str2 = u'{0:<%s}|' % (max_width - wide_chars(s2))
s3 = u'新年快乐'
fmt_str3 = u'{0:<%s}|' % (max_width - wide_chars(s3))
print(fmt_str1.format(s1))
print(fmt_str2.format(s2))
print(fmt_str3.format(s3))
```

`F`表示宽字符，如`￥`；`W`表示其它的字符，如中文汉字和符号。具体看 [EAST ASIAN WIDTH](http://www.unicode.org/reports/tr11/)

（注：先前以为宽字符都是全角字符，不过发现半角中文逗号也是属于 F，懒得研究这块了）

上面函数返回的是一个 unicode 字符串中，所有中文字符的个数，因为大部分中文字符在 utf-8 下是占三个字节，而宽度是两个字符，所以减去中文个数就是字节和宽度一样。注意这里是用的 unicode 字符串，如果在 Python2.x 下使用 str，则需要改为 `+ wide_chars()`。

当然上面还不是很严谨，因为没有具体研究哪些字符可能不是 3 个字节。[Python 中计算字符宽度](http://likang.me/blog/2012/04/13/calculate-character-width-in-python/) 这篇文章里提到了另外一种方案。不过目前使用来说，上面的例子已经足够了。


然后就是开始字符编码的各种风暴了……

## Terminal Emulator Character Encoding and $LANG

比如 Mac 下我使用的 Terminal 是 iTerm2, 终端的字符编码设置在`Profiles -> Terminal -> Character Encoding`，选择的是`Unicode (UTF-8)`。

`$LANG`设置是`zh_CN.UTF-8`, `zh_CN` 表示语言，即这里的中文，`UTF-8` 也就是编码集。终端显示的一些结果，如`ls`, `date`的输出、vim 的提示都是中文。


## locale 相关

关于`locale`输出各字段的含义，在`man 7 locale`中有解释。

对于区域设置，有两个特殊的：`C` 和 `POSIX`。

比如查看 mac 手册时，因为字符集原因，呈现的是乱码，这时经常会使用`LANG=C man xxx`，意思就是去掉本地化，使用程序自身的语言去呈现，当然一般就是英文了。

* [What does “LC_ALL=C” do?](http://unix.stackexchange.com/questions/87745/what-does-lc-all-c-do)
* [LC_ALL=C 的含义](http://blog.chinaunix.net/uid-74180-id-2055792.html)

关于 locale 的字段，比较常用的是`LC_ALL`, `LANG`, `LC_CTYPE`, `LC_TIME`等：

* `LC_ALL` 用于最高优先级的全局设置，一般为空；为空时会寻找相应字段`LC_*`的值，如果这个也没设置，就找默认的`LANG`值。
* `LC_CTYPE` 用于字节序列编码的解释，即字符显示等作用
* `LC_TIME` 时间的显示格式和本地化

参考：

* [Ubuntu wiki - locale](http://wiki.ubuntu.org.cn/Locale)
* [Ubuntu 国际化与本土化](http://wp.fungo.me/linux/ubuntu-i18n-l10n.html)


## 再说 Python Source Code Encodings

在 [Python Encoding]({{ site.baseurl }}{% post_url 2015-01-21-python-encoding %}) 提到过这个，这里再补充下。

这个的作用就是让 Python 解释器在读文件时，对里面的非 ascii 字符做编解码处理。

比如最常用的：

```python
coding=utf-8
```

如果设置其它的字符集，如 gb2312：

```python
coding=gb2312
```

则要求文件的编码是 gb2312，否则比如保存为 utf-8 编码，则以 gb2312 编码去读取文件会导致错误。

另外就是执行的输出，比如：

```python
#coding=gb2312
a = "中文"
print(a)
print(a.decode('gb2312').encode('gb2312'))
```

需要设置终端编码是 gb2312，否则输出是乱码。


## Python sys.getdefaultencoding()

Python 2.x 下，默认是 ascii，这个与系统环境无关。

在 [Python Encoding]({{ site.baseurl }}{% post_url 2015-01-21-python-encoding %}) 也提到了。

再举个例子，参考 [Python - Default Encoding](https://wiki.python.org/moin/DefaultEncoding)：

```python
#coding=utf-8
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
print sys.getdefaultencoding()
a = '中文' + u'abc'
```

如果不改变默认编码为 utf-8，则报错：

> UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0: ordinal not in range(128)

因为这个字符串拼接的实际效果是：

```python
a = '中文'.decode(sys.getdefaultencoding()) + u'abc'
```


## In vim, encoding vs fileencoding vs fileencodings

注意 fileencoding 和 fileencodings 区别，一个是单数，一个是复数。

* `encoding`：字符串。影响 buffer, register 等，就是在输入字符时最后呈现的样子。默认是"latin1" or value from $LANG
* `fileencoding`：字符串。新建文件写文件时的字符编码。如果没设置则和`encoding`一样，否则在写文件时会做转码；读取文件时，会被`fileencodings`设置。
* `fileencodings`：列表。当准备编辑一个已存在的文件时，vim 尝试使用第一个字符编码去读取文件，如果出错则使用第二个。

即：

* 读文件：从 fileencoding 转换到 encoding
* 写文件：从 encoding 转换到 fileencoding

所以在打开或保存时可以看到`converted`的标记。

举个例子：

```vim
set encoding=utf-8
set fileencoding=gb2312
set fileencodings=utf-8,gb2312
```

新建一个文件，vim 显示的编码使用 utf-8，**在输入中文后**，保存会看到左下角提示`converted`，命令行通过`file -i filename`可以看到文件字符集为 charset=iso-8859-1。再次打开文件时，通过 fileencodings 来设置 fileencoding，并进行转码到 encoding 设置。

延伸阅读：

* [Character Encoding Tricks for Vim](https://spin.atomicobject.com/2011/06/21/character-encoding-tricks-for-vim/)
* [用 vim 打开后中文乱码怎么办？](https://www.zhihu.com/question/22363620)

另外有个疑问 TODO：

* 如果我的 encoding 和 fileencoding 都设置的 gb2312，而终端设置的 UTF-8 时，实际保存的是 utf-8；如果终端是 gb2312，则保存的是 gb2312。
* 而如果 encoding 是 utf-8, fileencoding 是 gb2312, 终端是 UTF-8 时，实际保存的是 gb2312

这块目前还不清楚原因，只能猜测是因为终端字符集和 encoding buffer 这块有关，虽然设置的 gb2312，但是实际是终端的 utf-8，但是因为和 fileencoding 一样设置的 gb2312，所以不做转码。


## 总结一下

* 万能的 UTF-8 大法好
* 不要设置不一致，从终端到程序配置都统一为 UTF-8


## 其它参考

* [UTF-8 and Unicode FAQ for Unix/Linux](https://www.cl.cam.ac.uk/~mgk25/unicode.html#utf-8)
* [Effect of $LANG on terminal](http://unix.stackexchange.com/questions/48689/effect-of-lang-on-terminal)
* [浅析 Linux 的国际化与本地化机制](https://www.ibm.com/developerworks/cn/linux/l-cn-linuxglb/)
