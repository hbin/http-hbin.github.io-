---
layout: post
title: Ruby ElasticSearch Connection Pool
date: 2016-11-17 10:25:55 +0800
comments: true
categories: [Web]
tag: [ElasticSearch, Ruby]
---

由于 Elasticsearch 官方所有非 Java 的包都是通过 [HTTP](https://www.elastic.co/blog/found-interfacing-elasticsearch-picking-client#http-clients) 的方式链接的。
所以选用一个更快的，可以保持连接(Keep-Alive) 的 HTTP clint 非常重要。

这里有一份 Ruby 各种 HTTP client adapters 的对比测试，这篇博文非常详尽：http://reevoo.github.io/blog/2014/09/12/http-shooting-party/

结果如下图：

最快的 Curb 和 Excon 相差有 12 倍之巨！Patron 和 Excon 也相差 6.8 倍！

感谢 Elasticsearch-ruby 作者，已经默默为我们做了这部分工作，所以，我们需要做的只是：`gem install patron`:

[如下代码](https://github.com/elastic/elasticsearch-ruby/blob/bdf5e145e5acc21726dddcd34492debbbddde568/elasticsearch-transport/lib/elasticsearch/transport/client.rb#L196-L209)：

``` ruby
# Auto-detect the best adapter (HTTP "driver") available, based on libraries
# loaded by the user, preferring those with persistent connections
# ("keep-alive") by default
#
# @return [Symbol]
#
# @api private
#
def __auto_detect_adapter
  case
  when defined?(::Patron)
    :patron
  when defined?(::Typhoeus)
    :typhoeus
  when defined?(::HTTPClient)
    :httpclient
  when defined?(::Net::HTTP::Persistent)
    :net_http_persistent
  else
    ::Faraday.default_adapter
  end
end
```

这里最后一个 [Faraday](https://github.com/lostisland/faraday) 是一个 Ruby HTTP 请求的通用接口。它将各种常用 HTTP client 包的接口抽象出来，只要替换 Adapter 就可以实现不同库无缝切换。

但是这还不够！因为除了 Keep-Alive 还有一个更重要的池化（Pooling）！
从 Faraday 的 Patron Adapater [源码](https://github.com/lostisland/faraday/blob/master/lib/faraday/adapter/patron.rb#L73-L78)，

``` ruby
# https://github.com/lostisland/faraday/blob/master/lib/faraday/adapter/patron.rb#L73-L78

def create_session
  session = ::Patron::Session.new
  session.insecure = true
  @block.call(session) if @block
  session
end
```

可以看到，这里只是简单新建一个链接，所以我们可以把这个连接池化(Pooling)。[池化技术](https://en.wikipedia.org/wiki/Connection_pool)是减少计算机某些昂贵开销的一种有效手段。
网络请求就是典型的一种情况。将 Keepalive 的连接池化，可以有效提高性能。Ruby 里我们可以使用 [connection_pooling](https://github.com/mperham/connection_pool) 包，
所以一个简单的实现可以是：

``` ruby
# NOT TESTED, be caution to use it in production!

def create_session
  session = ConnectionPool::Wrapper.new(size: 5, timeout: 3) { ::Patron::Session.new }
  session.insecure = true
  @block.call(session) if @block
  session
end
```

当然，用 ConnectionPool::Wrapper 的方式相比 `with` 方式要慢一些，更好的的方式应该把 `call` 方法里对 session
的调用都用 with 包裹起来，参见 [connection_pooling](https://github.com/mperham/connection_pool) 的文档。
