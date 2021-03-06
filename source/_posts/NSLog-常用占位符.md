---
title: NSLog 常用占位符
date: 2020-09-18 14:59:08
tags:
 - C/OC 输出占位符
categories:
 - iOS
---


|符号|含义|
|:--:|:--|
|%@|Object-C 对象|
|%zu|size_t|
|%p|十六进制形式的指针地址|

----

|符号|含义|
|:--:|:--|
|%d、%i|十进制整数, 正数无符号, 负数有 "-" 符号|
|%zd|NSInteger|
|%tu|无符号NSUInteger|
|%u|十进制无符号整数|
|%o|八进制整数|
|%x %X|十六进制无符号整数, 没有 0x 前缀|

----

|符号|含义|
|:--:|:--|
|%c|字符|
|%C|unichar|
|%s|C 字符串|
|%.*s|Pascal 字符串|

----

|符号|含义|
|:--:|:--|
|%f|小数形式输出浮点数, 默认 6 位小数|
|%e|科学计算形式输出浮点数, 默认 6 位小数|
|%g|自动选择 %e 或者 %f|

----

|符号|含义|
|:--:|:--|
|l|在整型和浮点型之前, %d %o %x %u %f %e %g 代表长整型和长字符串|
|n(任意整数)|%8d 代表输出 8 位数字；%04d 代表输出 8 位整数，不够 8 位时高位自动补零，如 0002。输出总位数|
|.n|浮点数限制小数位数, %5.2f 表示 5 位数字 2 位小数, 如果是字符串，表示截取字符个数|
|-|字符左对齐|

