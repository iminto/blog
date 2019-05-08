---
title: PHP的mb_check_encoding函数的存在是鸡肋吗
date: 2018-01-12 21:24:23
tags: [PHP]
---

  前不久,有人问到我一个问题，就是使用mb_check_encoding来侦测一段字符的编码，预期是GBK编码，但是PHP给出来UTF-8编码的错误判断。那么，mb_check_encoding的正确姿势是什么呢？
我们来看一段代码，
```php
<?php
$utf8Str = '别abc扯淡';
var_dump(mb_check_encoding($utf8Str, 'UTF-8'));  //输出true
var_dump(mb_check_encoding($utf8Str, 'gbk')); //输出true
```
    这段代码的输出是啥呢？按理，我们的PHP文件保存为什么编码，那它输出的就应该是啥编码，然而以上输出的都是true。再换个例子，这样呢？
```php
<?php
$utf8Str = '别abc扯淡啊';
var_dump(mb_check_encoding($utf8Str, 'UTF-8'));
var_dump(mb_check_encoding($utf8Str, 'gbk'));
```
    后面多加了一个汉字，这次PHP做出了正确的判断，给出了是UTF-8的判断。那么mb_check_encoding到底有没有用？是这个函数有bug还是我自己不懂姿势？

    难道是，只要汉字是3的整数倍就会判断失灵？试验后确实是的，当然这只是表面现象，但无疑说明这个函数是不可靠的。为什么呢？其实原理说起来也不难理解，计算机并不懂什么叫乱码。一段文字，解释成UTF8或GBK其实都是可以的，我们用肉眼看到有了乱码，根据我们的经验，觉得解释成这种编码是错误的，而解释成另外一种编码才算正确。可是计算机不懂啊，你觉得有个字符很奇怪，你不认识所以认定是乱码，可计算机认识啊，它不觉得奇怪。除非字节数解释成另外一种编码，会多出一个字节，并且ASCII码也不是常见范围，计算机才能大胆判定解释成这种编码不对。所以这样去检测编码是无法完全可靠地.

    那既然mb_check_encoding这个函数不可靠，那么用正则可靠么？或许吧。
    但是我们更应该关注的是PHP为什么会有这么一个功能？为什么其他语言没有这个方法，或者根本不会遇到这个问题？

    问题还是出在PHP本身。因为客户端可能会有多种编码输入，PHP为了解决这个问题就引入这么一个贴心的函数给使用者。可是PHP不应该是遇到问题就去动歪脑筋解决问题啊，而且规范问题。为什么其他语言不需要在SDK里引入这个方法呢？或者说是PHP程序员的使用姿势不正确?

    最后，其实PHP给出这个函数也不算错，但是一定要参照其他语言里的惯行做法，在文档里说清楚，这个函数的判断的是一种“可信度”，而不是给出一个非此即彼的“权威”结果。但是遗憾的是，这个函数的文档里没说很好的说清楚，而是这么写的，
> “Checks if the specified byte stream is valid for the specified encoding. It is useful to prevent so-called "Invalid Encoding Attack Returns TRUE on success or FALSE on failure.”

    其实加上这样一句话“This function only give the confidence level of the result”就好了，也就不会平白引起那么多的疑虑。

    比如，Python的做法就比较专业，[chardet](https://pypi.python.org/pypi/chardet/)模块给出的是一个置信检测，而不是非true即false的判断。java里面的第三方工具包[cpdetector](http://cpdetector.sourceforge.net/index.shtml)也指出了其规则，按照“谁最先返回非空的探测结果，就以该结果为准”的原则返回探测到的字符集编码。其是基于统计学原理的，不保证完全正确。
