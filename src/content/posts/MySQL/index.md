---
title: MySQL
published: 2024-01-24
description: MySQL学习笔记
image: ./cover.jpg
category: SQL
tags: [MySQL]
draft: false
---

### MySQL的执行顺序

`from` > `join` > `where` > `group by` > 聚合函数 > `having` > `select` > `order by` > `limit`

> 格式化函数(格式化数字：FROMAT(),格式化日期：DATE_FORMAT())会在`group by`之前执行

### count的条件用法

`count` 函数用于计算非 `null` 值的数量。

`count(age > 20 or null)`里面的`or null`必须加，否则就等于`count(*)`

`count`对于不管是0还是1，都会计数一次，只有`null`不会被计数。

- `age > 20`：这是一个布尔表达式，当 `age` 大于 20 时返回 `TRUE`，否则返回 `FALSE`。
- `or null`：当布尔表达式 `age > 20` 为 `FALSE` 时，将其转换为 `null`，从而使得这些记录不会被 `count` 计数。

或者可以写为`sum(age > 20)`
