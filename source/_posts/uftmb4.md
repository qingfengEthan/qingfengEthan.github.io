---
title: Mysql的utf8与utf8mb4区别，utf8mb4_bin、utf8mb4_general_ci与utf8mb4_unicode_ci的选择
date: 2019-7-18 10:27:06
tags:
---

------
#### utf8 与 utf8mb4
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 标准的 UTF-8 字符集编码是可以用 1~4 个字节去编码21位字符，是一种变长的编码格式，这几乎包含了是世界上所有能看见的语言了。然而在MySQL里实现的utf8最长使用3个字节，节省空间但不能表达全部的UTF-8，只支持到了 Unicode 中的“基本多文种平面”（U+0000至U+FFFF，Basic Multilingual Plane，BMP），包含了控制符、拉丁文，中、日、韩等绝大多数国际字符，但并不是所有，最常见的就算现在手机端常用的表情字符 emoji和一些不常用的汉字，如 “墅” ，这些需要四个字节才能编码出来。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL在 5.5.3 之后增加了 utf8mb4 字符编码，mb4即 most bytes 4,使用4个字节来表示完整的UTF-8。简单说 utf8mb4 是 utf8 的超集并完全兼容utf8，能够用四个字节存储更多的字符。

注：QQ里面的内置的表情不算，它是通过特殊映射到的一个gif图片。一般输入法自带的就是。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当你的数据库里要求能够存入这些表情或宽字符时，可以把字段定义为 utf8mb4，同时要注意连接字符集也要设置为utf8mb4，否则在 严格模式 下会出现 Incorrect string value: /xF0/xA1/x8B/xBE/xE5/xA2… for column 'name'这样的错误，非严格模式下此后的数据会被截断。


#### utf8mb4_bin、utf8mb4_unicode_ci 与 utf8mb4_general_ci
**utf8mb4_bin：** 将字符串每个字符用二进制数据编译存储，区分大小写，而且可以存二进制的内容。

**utf8mb4_general_ci**：ci即case insensitive，不区分大小写。是一个遗留的 校对规则，不支持扩展，它仅能够在字符之间进行逐个比较，没有实现Unicode排序规则，在遇到某些特殊语言或者字符集，排序结果可能不一致。但是，在绝大多数情况下，这些特殊字符的顺序并不需要那么精确。

**utf8mb4_unicode_ci**：是基于标准的Unicode来排序和比较，能够在各种语言之间精确排序，Unicode排序规则为了能够处理特殊字符的情况，实现了略微复杂的排序算法。


**collate规则：**<br>
utf8mb4_bin 大小写敏感<br>
utf8mb4_general_cs 大小写敏感<br>

 *_bin: 表示的是binary case sensitive collation，也就是说是区分大小写的<br>
 *_cs: case sensitive collation，区分大小写<br>
 *_ci: case insensitive collation，不区分大小写<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mysql默认的字符检索策略：utf8_general_ci，表示不区分大小写；utf8_general_cs表示区分大小写，utf8_bin表示二进制比较，同样也区分大小写 。（注意：在Mysql5.6.10版本中，不支持utf8_genral_cs！）
```
utf8mb4_general_ci           P=p  Q=q  R=r=Ř=ř   S=s=ß=Ś=ś=Ş=ş=Š=š  sh  ss    sz
utf8mb4_unicode_ci           P=p  Q=q  R=r=Ř=ř   S=s=Ś=ś=Ş=ş=Š=š    sh  ss=ß  sz
```

**如何选择**<br>
字符除了需要存储，还需要排序或比较大小，涉及到与编码字符集对应的 排序字符集（collation）。ut8mb4对应的排序字符集常用的有 utf8mb4_unicode_ci、utf8mb4_general_ci
主要从排序准确性和性能两方面看：
- **准确性**<br>
  utf8mb4_unicode_ci： 是基于标准的Unicode来排序和比较，能够在各种语言之间精确排序。<br>
utf8mb4_general_ci： 没有实现Unicode排序规则，在遇到某些特殊语言或字符是，排序结果可能不是所期望的。
但是在绝大多数情况下，这种特殊字符的顺序一定要那么精确吗。比如Unicode把ß、Œ当成ss和OE来看；而general会把它们当成s、e，再如ÀÁÅåāă各自都与 A 相等。
- **性能**<br>
  utf8mb4_general_ci： 在比较和排序的时候更快<br>
utf8mb4_unicode_ci： 在特殊情况下，Unicode排序规则为了能够处理特殊字符的情况，实现了略微复杂的排序算法。
但是在绝大多数情况下，不会发生此类复杂比较。general理论上比Unicode可能快些，但相比现在的CPU来说，它远远不足以成为考虑性能的因素，索引涉及、SQL设计才是。<br> 
这也从另一个角度告诉我们，不要可能产生乱码的字段作为主键或唯一索引。例如：以url来作为唯一索引，但是它记录的有可能是乱码。

总结：utf8mb4_general_ci 更快，utf8mb4_unicode_ci 更准确。推荐是 utf8mb4_unicode_ci，将来 8.0 里也极有可能使用变为默认的规则。相比选择哪一种collation，使用者更应该关心字符集与排序规则在db里需要统一。

参考：<br>
&nbsp;&nbsp;&nbsp;&nbsp;[http://seanlook.com/2016/10/23/mysql-utf8mb4/](http://seanlook.com/2016/10/23/mysql-utf8mb4/)
&nbsp;&nbsp;&nbsp;&nbsp;[https://my.oschina.net/xsh1208/blog/1052781](https://my.oschina.net/xsh1208/blog/1052781/)
