## RDB

### 什么是 RDB 文件？

rdb 文件是一种压缩的二进制格式文件，是 redis 在某一时刻的快照，如果 redis 宕机，可以根据 rdb 文件进行恢复

### 如何生成 RDB 文件？

save 和 bgsave

save 会导致服务器阻塞，所有命令被拒绝

bgsave 会 fork 一个子进程保存 rdb 文件，服务器正常接收命令并处理，但有几个命令除外

* save
* bgsave
* bgrewriteaof

执行 bgsave 时会拒绝 save 命令和 bgsave 命令，对 bgrewriteaof 命令有不同表现

* 如果先执行的 bgsave，bgrewriteaof 会等到 bgsave 执行完毕之后再执行
* 如果先执行的 bgrewriteaof，bgsave 会被拒绝

原因就是性能考虑，两个命令都是重 io 操作，同时执行影响太大，而且，两个都是做持久化，功能是一样的，没必要重复

### 如何载入 RDB 文件？

启动时自动载入，但是优先载入 aof 文件，只有 redis 没有开启 aof 的时候才会加载 rdb 文件

因为 aof 文件的更新频率更高

### 自动间隔性保存

可以设置多个条件，任意一个条件满足都会执行 gbsave 命令

可以通过配置文件指定，也可以通过启动参数指定

配置格式如下

```
save 900 1 # 900 秒内至少修改了 1 次
save 300 10 # 300 秒内至少修改了 10 次
save 60 100# 60 秒内至少修改了 100 次
```

**如何实现的呢？**

dirty 计数器和 lastsave 时间戳

>《Redis 设计与实现》在讲这段的时候有点模糊，作者说 lastsave 会被更新为 bgsave 执行成功的时间戳，那么 dirty 计数器怎么更新呢，因为 bgsave 执行时服务器仍然在接收命令，如果 lastsave 更新为 bgsave 执行完成的时间，那么 bgsave 执行期间的修改就不会被记录，经过我查看代码，确认了这个问题

>在执行 bgsave 之前，redis 会记录下当前的 dirty 值，叫做 dirty_before_bgsave，在 bgsave 完成后，时间戳更新为当前时间，并且将 dirty 值更新为当前 dirty 减去 dirty_before_bgsave 的值，这样现在 dirty 值就变成了bgsave 执行期间 redis 执行的修改数量，这段逻辑可以在 `rdb.c/
rdbSaveBackground` 和 `rdb.c/backgroundSaveDoneHandlerDisk` 两个函数中看到


