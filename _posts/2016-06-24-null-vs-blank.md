---
layout: post
title: "NULL vs Empty String"
date: 2016-06-24 16:09:16 +0800
comments: true
categories: [Web]
tags: [Rails, Database]
---
在设计表的时候，一般字段默认值都会是 NULL，但是，我们在使用的时候，会碰到这个问题：用户提交一个表单值后，原本默认值是 NULL 的字段被赋上了空字符串！

## NULL 还是 Empty ？

首先来看看这两种值的含义：

* NULL -> 没有值
* 空字符串 -> 有值，但值为 ""

比如，邮箱字段：用空字符串就是有问题的，因为，空字符串本身就是一个不合法的邮箱地址！所以，如果你的场景中 NULL 和 空字符串的定义不一样，那就得考虑如何处理上面遇到的情况了。

## 数据库怎么定义？

1. 如果一个字段是 NOT NULL，那么我的就可以认定这是一个必填项，那确保给他一个默认值。
2. 如果一个字段是 ALLOW NULL，那么不要给他赋一个空的字符串。

## 代码中怎么处理？

对于允许为 NULL 的字段，我们应该将 form 表单提交上来的空字符串换成 NULL 值：

``` ruby
attributes.each do |column, value|
  self[column].present? || self[column] = nil
end
```

已经有一些 Gem 可以处理这个问题，比如 [attribute_normalizer](https://github.com/mdeering/attribute_normalizer) 只需要在 Model 中声明：`normalize_attributes :email` 它会默认执行 strip 和 blank 两个标准化方法。

<br />

---

_**[References]**_

* http://stackoverflow.com/questions/405909/null-vs-empty-when-dealing-with-user-input/405945#405945
* http://programmers.stackexchange.com/questions/32578/sql-empty-string-vs-null-value
* http://stackoverflow.com/questions/7202319/rails-force-empty-string-to-null-in-the-database
