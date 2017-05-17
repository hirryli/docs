[http:\/\/www.cnblogs.com\/lijiasnong\/p\/5478080.html](http://www.cnblogs.com/lijiasnong/p/5478080.html)

memcached LRU算法

http:\/\/blog.csdn.net\/fusan2004\/article\/details\/51705904

Memcached的LRU几种策略 1. 惰性删除。memcached一般不会主动去清除已经过期或者失效的缓存，当get请求一个item的时候，才会去检查item是否失效。 2. flush命令。flush命令会将所有的item设置为失效。 3. 创建的时候检查。Memcached会在创建ITEM的时候去LRU的链表尾部开始检查，是否有失效的ITEM，如果没有的话就重新创建。 4. LRU爬虫。memcached默认是关闭LRU爬虫的。LRU爬虫是一个单独的线程，会去清理失效的ITEM。

http:\/\/www.jianshu.com\/p\/1bfce103bf74



memcache hash规则

http:\/\/www.oschina.net\/question\/565065\_79795

