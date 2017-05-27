---
layout: post
title: Python与编码
category: 编程语言
keywords: Python , 2017, 编程语言
---

*在写豆瓣爬虫的时候遇到了一些莫名其妙的关于编码的问题，于是稍微学习了一下*

# 编码简介
## ASCII(American Standard Code for Information Interchange)
单字节的编码，表示英文字母和各种其他符号，但依然只用到了一半(\x80以下)

## MBCS(Multi-Byte Character Set)
IBM 发明了Code Page 的概念，将各种编码整合在一起。例如GBK是第936页，也就是CP936。
MBCS便是这些编码的总称。Windows里根据你设定的区域不同，MBCS指代不同的编码。如在简体中文的Windows 中，就会变成GBK

## Unicode
所有语言都用同一套字符集表示。最初的标准UCS-2使用两个字节表示一个字符。后来出现的UCS-4标准，使用四个字节表示一个字符。在计算机内存中统一使用Unicode

## UTF-xx
UCS(Unicode Character Set)只是字符对应码位的一张表。字符具体如何传输和储存则是由UTF(UCS Transformation Format)来负责。
* UTF-16: 直接用UCS 的码位来表示。
* UTF-8: 变长的编码形式，兼容ASCII。如中文使用三个字节保存。

# 关于python 2.7的字符串编码

python2.7的字符串编码比较乱，暂且举几个例子：
* 在所有情况下，使用u'xxx' 的字符串都为Unicode，类型也为Unicode
* 在Windows的cmd(包括Python命令行) 或文件中，使用r'xxx' 或直接 'xxx' 的字符串为输入环境的编码方式，cmd 即GBK，类型为str
* 使用encode() 将一个unicode 对象转换为str 对象
* 使用decode() 将一个str 对象转换为unicode 对象
* 在print 时，unicode使用默认编码解码为str。str直接输出，不需解码

*额外的发现：在ipython里面，默认编码是utf-8,使用gbk会乱码。看来是跟Linux保持一致的*


# 小结
* cmd 中使用GBK编码
* print 字符串时会自动由Unicode 转换为str
* 但使用print soup 时，soup 会调用自身的__str__，默认编码为UTF-8，所以直接在cmd中输出会乱码
* 尽量在处理字符串内容前都转换为Unicode，随后统一转换为UTF-8 或GBK
