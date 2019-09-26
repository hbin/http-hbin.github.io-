---
layout: post
title: "[译] How does the Hash works in Ruby"
date: 2017-01-14 18:44:58 +0800
comments: true
categories: [Ruby]
tag: [Algorithm]
---
原文：https://launchschool.com/blog/how-the-hash-works-in-ruby

这篇文章将会简短的介绍 Hash 这种数据结构，它是如何实现的，以及 MRI Ruby Hash 的演变历史。

## 什么是 Hash？

Hash 是一种键-值（key-value）存储结构，它也被称作字典（译者注：Python 就是 dict）或者关系型数组（Associative array）。得益于这种键-值的存储结构，它可以进行高效的插入和查找操作，而且时间复杂度仅是 O(1)。这种特性使得 Hash 成为程序员最有用的编程工具之一，所以绝大多数的编程语言都把它纳入到核心库。

在 Ruby 中，你可以用字面量的方式声明一个 Hash，如：

``` ruby
h = { color: 'black', font: 'Monaco' }
h = { :color => 'black', :font => 'Monaco' }
```

或者，用传统的 new 方法：

``` ruby
h = Hash.new
h[:color] = 'black'
h[:font] = 'Monaco'
```

## Hash 是如何存储数据的？它为什么高效？

为了理解这个问题，让我们先回顾下基本的线性存储结构 - 数组（Array）。如果我们知道其中某个元素的下标，那我们就可以快速的随机访问它所存储的元素。

``` ruby
a = [1, 2, 4, 5]
puts a[2] #> 4
```

如果 Hash 中存储的键是整数值，并且这个值也比较小，比如它在 1-20 或者 1-100 这个范围内，那我们可以简单的用数组来存，把键作为数组的下标。

举个例子，我们需要存储这样一组数据：

* 一个教室有 20 个学生，我们要存这个教室里所有学生的名字
* 每个学生的 ID 都在 1 - 20 之间
* 任意两个学生的 ID 都不相同

我们就可以简单的用数组来存，假设数据如下：


| Key  | Value        |
| ---- |:------------:|
| 1    | Belle        |
| 2    | Ariel        |
| 3    | Peter Pan    |
| 4    | Mickey Mouse |


那数组就可以表示为：

```
students = ['Belle', 'Ariel', 'Peter Pan', 'Mickey Mouse']
```

但是，如果学生的 ID 是 4 位数呢？那我们只能用一个长度为 10000 的数组来存，这样才能通过 ID 来访问学生的名字。（很显然，这浪费了很多空间），为了解决这个问题，我们可以用 4 位数 ID 的后两位来表示其在数组中的位置（比如，我们有个学生 ID 是 4221，它的后两位是 21）。但是假设现在，我们有另外一个学生，它的 ID 是 3221，它的后两位也是 21，那我们只能把这两个数放到数组的同一个下标内，这就会造成数据冲突。如：

| Key        | Hash(key) = last 2 digits | Value        |
| ---------- |:-------------------------:|:------------:|
| 4221, 3221 | 21                        | Belle, Sofia |
| 1357       | 57                        | Ariel        |
| 4612       | 12                        | Peter Pan    |
| 1514       | 14                        | Mickey Mouse |


``` ruby
students = Array.new(100)
students[21] = ['Belle', 'Sofia']
students[57] = 'Ariel'
students[12] = 'Peter Pan'
students[14] = 'Mickey Mouse'
```

而且，如果 ID 是 10 位数呢？或者 ID 是字符串呢？这就种方式就变得不高效或者不适用了。所以，太简单的哈希策略会导致很多问题。

## 那 Ruby 的哈希策略是什么呢？

好了，我们现在已经明白哈希策略的目的就是为了把某个键转化为有限范围内的一个整数。为了缩小这个范围，一个通常的策略就是取模运算。取模运算策略，就是把键去除以存储表的大小，得到的余数就是这个键在存储表中的位置。因此，在上面这个例子中，如果存储表的大小为 20，取模之后他们的位置分别是1, 17, 12, 14，运算如下：

* 4221 % 20 = 1
* 1357 % 20 = 17
* 4612 % 20 = 12
* 1514 % 20 = 14

但是，在真实的编程过程中，键并不总会是整数，他们也可能是字符串，对象，或者其他数据类型。这个时候，我们可以用单向的哈希函数（digest）对键求值，然后再用取模运算得出其存储位置。这种哈希函数是一种数学函数，它可以接收任意长度的字符串，然后返回一个固定长度的整数值。这也正是 Hash 这种数据结构的名字由来。Ruby 使用了一种叫做 [MurMurHash](https://en.wikipedia.org/wiki/MurmurHash) 的非加密型哈希函数，求得的值再用质数 M 取模，M 的取值根据存储结构大小来判断。

``` c
murmur_hash(key) % M
```

这段代码可以在 Ruby 的源代码里找到 [st.c file](https://github.com/ruby/ruby/blob/1b5acebef2d447a3dbed6cf5e146fda74b81f10d/st.c)。

有的时候，两个不同的键哈希后会得到同样的值，所以他们的值会被存放到表中的相同位置（译者注：也叫做桶），这种情况被称作哈希冲突（hash collision）。

## Ruby 是如何处理哈希冲突的？

哈希函数面临的一个问题是如何让值更加分散。如果大多数键哈希取模后都落到同一个桶里，那我们再要找某个 key 时，只能先找到这个桶，然后遍历整个桶里的数据，最终才找到匹配的数据。这种方式就违背了我们设计 Hash 这种数据结构的初衷，也不能在 O(1) 的时间复杂度里随机访问了。由于需要遍历，此时的时间复杂度已近降为 O(n) 了。

研究发现，如果上面的除数 M 是一个质数的话，其结果会更加均匀分布。但是，即使选择了最合适的除数，当数据量增加时，冲突仍然不可避免。Ruby 会根据当前数据的密度（density）调整除数 M 的值。这里的密度指存储表中某一个桶里数据的个数。在上面的例子中，其密度为 2，因为第一个桶里有两个值。

Ruby 默认设置允许的最大密度为 5

``` c
#define ST_DEFAULT_MAX_DENSITY 5
```

当密度达到 5 的时候， Ruby 就会调整除数 M，重新计算所有的记录的哈希值，调整其位置。Data Stuctures using C 说：“求除数 M 的算法是：生成最为接近 2 的幂数的质数”。源码 st.c 中第 158 行计算了新 Hash 大小。


``` c
new_size(st_index_t size)
{
    st_index_t i;
    for (i=3; i<31; i++) {
       if ((st_index_t)(1<<i) > size) return 1<<i;
    }
    return -1;
}
```

JRuby 的实现可能会更容易理解一些，它直接预生成了这些质数数组。如下数组，它有 11，19 等等等等

``` java
private static final int MRI_PRIMES[] = { 8 + 3, 16 + 3, 32 + 5, 64 + 3, 128 + 3, 256 + 27, ....};
```

当 Hash 的大小持续增长，达到一定值的时候，重新哈希（rehashing）可能会导致性能问题（Spike performance）。Pat Shaughnessy 在它的 <[Ruby under a Microscope](http://patshaughnessy.net/ruby-under-a-microscope)> 这本书中详细分析了这一情况，并且用图形化的方式向你展示了重新哈希导致的性能问题（译者注：Figure 7-8）。

## Ruby 的 Hash 在每一个进程里都不一样

一个有意思的事是，Hash 在 Ruby 各个进程都是唯一的。Ruby 会给 Murmur hash 算法一个随机的 seed，在不同的进程中相同的，即使 key 相同，哈希值也不一样。

## Ruby 2.0 以后会把长度小于等于 6 的 Hash 打包（packed）

另一个有意思的事是，当 Hash 数据非常少的时候（长度小于或等于 6），Ruby 会把这些元素打包起来只存到同一个桶中，而不是根据 Key 去哈希取模然后分布到不同的桶中。也就是说，它就又变回了简单的数组。这个特性是最近添加的，这个 [pull request](https://github.com/ruby/ruby/pull/84) 的作者在留言处写到：

> 调研表明，一个典型的 Rails 应用会分配大量的小的 Hash，其中最多 40% 的 Hash 大小不超过一个。

## Ruby 的 Hash 键是有序的

从 Ruby 1.9.3 开始，Hash 的键会按照其插入顺序排序。在 Ruby-forum 上有人问，为什么要让键有序，以及这会不会对性能造成影响。Ruby 的创始人 Matz 也在那里回复了，

> Q: 有人能解释下，为什么要加入这个特性吗？

> Matz: 在某些情况下非常有用，尤其是关键字参数。

> Q: 这不会拖慢在 Hash 上的方法调用吗？

> Matz: 不会的，对 Hash 操作并不会碰到其顺序的信息，除了遍历的时候。内存占用也只是增加一点点。

（译者注：这里略过两段关于 Ruby 2.0 引入的新特性介绍）...

## 结语

数据结构是相通的。无论是 Java，Python，还是 Ruby 他们的 Hash 的本质都是一样的。也希望，通过这边文章，能帮助大家从更深的角度去理解它，而不是仅仅把自己当做一个 API 的使用者，然后写出运行效率更高的代码，甚至能积极参与进来，帮助社区一起打造下一个 Ruby 版本。

译完

---

_**[References]**_

* https://en.wikipedia.org/wiki/Hash_table
* https://en.wikipedia.org/wiki/One-way_function
* https://www.amazon.com/Data-Structures-Using-Aaron-Tenenbaum/dp/0131997467
* https://github.com/ruby/ruby/blob/1b5acebef2d447a3dbed6cf5e146fda74b81f10d/st.c
* https://github.com/jruby/jruby/blob/master/core/src/main/java/org/jruby/RubyHash.java
* https://github.com/ruby/ruby/pull/84
