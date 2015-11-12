---
Title:   安装完 MySQL 后必须调整的 10 项配置
Summary: 当我们被人雇来监测MySQL性能时，人们希望我们能够检视一下MySQL配置然后给出一些提高建议。许多人在事后都非常惊讶，因为我们建议他们仅仅改动几个设置，即使是这里有好几百个配置项。这篇文章的目的在于给你一份非常重要的配置项清单。
Authors: Django Wong
Date:    2015-03-31
---

当我们被人雇来监测MySQL性能时，人们希望我们能够检视一下MySQL配置然后给出一些提高建议。许多人在事后都非常惊讶，因为我们建议他们仅仅改动几个设置，即使是这里有好几百个配置项。这篇文章的目的在于给你一份非常重要的配置项清单。

我们曾在几年前在博客里给出了这样的[建议](http://www.mysqlperformanceblog.com/2006/09/29/what-to-tune-in-mysql-server-after-installation/)，但是MySQL的世界变化实在太快了！

## 写在开始前…

即使是经验老道的人也会犯错，会引起很多麻烦。所以在盲目的运用这些推荐之前，请记住下面的内容：

- 一次只改变一个设置！这是测试改变是否有益的唯一方法。  
- 大多数配置能在运行时使用SET GLOBAL改变。这是非常便捷的方法它能使你在出问题后快速撤销变更。但是，要永久生效你需要在配置文件里做出改动。  
- 一个变更即使重启了MySQL也没起作用？请确定你使用了正确的配置文件。请确定你把配置放在了正确的区域内(所有这篇文章提到的配置都属于 [mysqld])  
- 服务器在改动一个配置后启不来了：请确定你使用了正确的单位。例如，`innodb_buffer_pool_size`的单位是MB而`max_connection`是没有单位的。  
- 不要在一个配置文件里出现重复的配置项。如果你想追踪改动，请使用版本控制。  
- 不要用天真的计算方法，例如”现在我的服务器的内存是之前的2倍，所以我得把所有数值都改成之前的2倍“。  

## 基本配置

你需要经常察看以下3个配置项。不然，可能很快就会出问题。

**`innodb_buffer_pool_size`**:这是你安装完InnoDB后第一个应该设置的选项。缓冲池是数据和索引缓存的地方：这个值越大越好，这能保证你在大多数的读取操作时使用的是内存而不是硬盘。典型的值是5-6GB(8GB内存)，20-25GB(32GB内存)，100-120GB(128GB内存)。

**`innodb_log_file_size`**：这是redo日志的大小。redo日志被用于确保写操作快速而可靠并且在崩溃时恢复。一直到MySQL 5.1，它都难于调整，因为一方面你想让它更大来提高性能，另一方面你想让它更小来使得崩溃后更快恢复。幸运的是从MySQL 5.5之后，崩溃恢复的性能的到了很大提升，这样你就可以同时拥有较高的写入性能和崩溃恢复性能了。一直到MySQL 5.5，redo日志的总尺寸被限定在4GB(默认可以有2个log文件)。这在MySQL 5.6里被提高。

一开始就把`innodb_log_file_size`设置成512M(这样有1GB的redo日志)会使你有充裕的写操作空间。如果你知道你的应用程序需要频繁的写入数据并且你使用的时MySQL 5.6，你可以一开始就把它这是成4G。

**`max_connections`**:如果你经常看到‘Too many connections’错误，是因为`max_connections`的值太低了。这非常常见因为应用程序没有正确的关闭数据库连接，你需要比默认的151连接数更大的值。`max_connection`值被设高了(例如1000或更高)之后一个主要缺陷是当服务器运行1000个或更高的活动事务时会变的没有响应。在应用程序里使用连接池或者在MySQL里使用[进程池](http://www.mysqlperformanceblog.com/2014/01/23/percona-server-improve-scalability-percona-thread-pool/)有助于解决这一问题。

## InnoDB配置

从MySQL 5.5版本开始，InnoDB就是默认的存储引擎并且它比任何其他存储引擎的使用都要多得多。那也是为什么它需要小心配置的原因。

**`innodb_file_per_table`**：这项设置告知InnoDB是否需要将所有表的数据和索引存放在共享表空间里（`innodb_file_per_table` = OFF） 或者为每张表的数据单独放在一个.ibd文件（`innodb_file_per_table` = ON）。每张表一个文件允许你在drop、truncate或者rebuild表时回收磁盘空间。这对于一些高级特性也是有必要的，比如数据压缩。但是它不会带来任何性能收益。你不想让每张表一个文件的主要场景是：有非常多的表（比如10k+）。

MySQL 5.6中，这个属性默认值是ON，因此大部分情况下你什么都不需要做。对于之前的版本你必须在加载数据之前将这个属性设置为ON，因为它只对新创建的表有影响。

**`innodb_flush_log_at_trx_commit`**：默认值为1，表示InnoDB完全支持ACID特性。当你的主要关注点是数据安全的时候这个值是最合适的，比如在一个主节点上。但是对于磁盘（读写）速度较慢的系统，它会带来很巨大的开销，因为每次将改变flush到redo日志都需要额外的fsyncs。将它的值设置为2会导致不太可靠（unreliable）因为提交的事务仅仅每秒才flush一次到redo日志，但对于一些场景是可以接受的，比如对于主节点的备份节点这个值是可以接受的。如果值为0速度就更快了，但在系统崩溃时可能丢失一些数据：只适用于备份节点。

**`innodb_flush_method`**: 这项配置决定了数据和日志写入硬盘的方式。一般来说，如果你有硬件RAID控制器，并且其独立缓存采用write-back机制，并有着电池断电保护，那么应该设置配置为O_DIRECT；否则，大多数情况下应将其设为fdatasync（默认值）。sysbench是一个可以帮助你决定这个选项的好工具。

**`innodb_log_buffer_size`**: 这项配置决定了为尚未执行的事务分配的缓存。其默认值（1MB）一般来说已经够用了，但是如果你的事务中包含有二进制大对象或者大文本字段的话，这点缓存很快就会被填满并触发额外的I/O操作。看看`Innodb_log_waits`状态变量，如果它不是0，增加`innodb_log_buffer_size`。


## 其他设置

**`query_cache_size`**: query cache（查询缓存）是一个众所周知的瓶颈，甚至在并发并不多的时候也是如此。 最佳选项是将其从一开始就停用，设置`query_cache_size` = 0（现在MySQL 5.6的默认值）并利用其他方法加速查询：优化索引、增加拷贝分散负载或者启用额外的缓存（比如memcache或redis）。如果你已经为你的应用启用了query cache并且还没有发现任何问题，query cache可能对你有用。这是如果你想停用它，那就得小心了。

**`log_bin`**：如果你想让数据库服务器充当主节点的备份节点，那么开启二进制日志是必须的。如果这么做了之后，还别忘了设置`server_id`为一个唯一的值。就算只有一个服务器，如果你想做基于时间点的数据恢复，这（开启二进制日志）也是很有用的：从你最近的备份中恢复（全量备份），并应用二进制日志中的修改（增量备份）。二进制日志一旦创建就将永久保存。所以如果你不想让磁盘空间耗尽，你可以用[PURGE BINARY LOGS](http://dev.mysql.com/doc/refman/5.6/en/purge-binary-logs.html) 来清除旧文件，或者设置`expire_logs_days`来指定过多少天日志将被自动清除。

记录二进制日志不是没有开销的，所以如果你在一个非主节点的复制节点上不需要它的话，那么建议关闭这个选项。

**`skip_name_resolve`**：当客户端连接数据库服务器时，服务器会进行主机名解析，并且当DNS很慢时，建立连接也会很慢。因此建议在启动服务器时关闭`skip_name_resolve`选项而不进行DNS查找。唯一的局限是之后`GRANT`语句中只能使用IP地址了，因此在添加这项设置到一个已有系统中必须格外小心。

## 总结

当然还有其他的设置可以起作用，取决于你的负载或硬件：在慢内存和快磁盘、高并发和写密集型负载情况下，你将需要特殊的调整。然而这里的目标是使得你可以快速地获得一个稳健的MySQL配置，而不用花费太多时间在调整一些无关紧要的MySQL设置或读文档找出哪些设置对你来说很重要上。

-----
翻译：<http://www.oschina.net/translate/10-mysql-settings-to-tune-after-installation>  
原文：<http://www.percona.com/blog/2014/01/28/10-mysql-settings-to-tune-after-installation/>
