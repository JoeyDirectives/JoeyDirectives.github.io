---
layout:       post
title:        "Adaptive Hash Index"
author:       "joey"
header-style: text
catalog:      true
tags:
    - MySQL
---
## Adaptive Hash Index概念

Adaptive Hash Index的概念上：在MySQL运行的过程中，如果InnoDB发现：

- 有很多寻路很长（比如B+树层数太多、回表次数多等情况）的SQL；
- 有很多SQL会命中相同的页面（Page）。

**InnoDB会在自己的内存缓冲区（Buffer Pool）里，开辟一块区域，建立自适应哈希索引（Adaptive Hash Index，AHI），以加速查询**。

- **Hash数据结构都是包含键（Key）、值（Value）的，**在Adaptive Hash Index，Key就是经常访问到的索引键值，Value就是该索引键值匹配的完整记录所在页面（Page）的位置。
- **因为是MySQL InnoDB自己维护创建的，所以称之为“自适应”哈希索引，但系统也有误判的时候，也不能起到加速查询的效果。**
- **Adaptive Hash Index是内存结构，非持久化；**
- **Adaptive Hash Index只能用于等值比较，例如=、<=>、IN、AND等，无法用于排序；**
- **Adaptive Hash Index是MySQL InnoDB自己维护创建的，人为无法干预。初始化为innodb_buffer_pool_size的1/64，会随着InnoDB Buffer Pool动态调整。**