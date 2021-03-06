## 持久化
Redis支持RDB和AOF两种持久化机制，持久化功能有效的避免因进程退出造成的数据丢失问题，
当下次重启时利用之前持久化的文件即可实现数据恢复

### RDB
RDB是把当前进程数据生成快照保存到硬盘的过程，触发RDB持久化过程分为手动触发和自动触发。
#### 手动触发
两种方式
- save：阻塞当前Redis服务器，直到RDB过程完成为止，对于内存比较大的实例会造成长时间
  阻塞，线上环境不建议使用
- bgsave：Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束

#### 自动触发
- 使用save相关配置，如"save m n"。表示m秒内数据集存在n次修改时，自动触发bgsave
- 如果从节点执行全量赋值操作，主节点自动执行bgsave生成RDB文件并发送给从节点
- 执行debug reload命令重新加载redis时，也会触发save操作
- 默认情况下执行shutdown命令式，如果没有开启AOF持久化动能则自动执行bgsave

#### 优缺点
- 优点
    - RDB是一个紧凑压缩的二进制文件，代表Redis在某个时间点上的数据快照。非常适用于
      备份，全量复制等场景
    - Redis加载RDB恢复数据远远快与AOF
- 缺点
    - RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork
      操作创建子进程，属于重量及操作，频繁执行成本过高
  
### AOF
AOF(Append Of File)，以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中
的命令达到恢复数据的目的。AOF的主要作用是解决了数据持久化的实时性，目前已经是Redis
持久化的主流方式。

AOF默认关闭，开启需要修改配置文件：appendonly yes。AOF文件名通过appendfilename
配置，默认文件名是appendonly.aof。

AOF的工作流程操作：命令写入(append)、文件同步(sync)、文件重写(rewrite)、
重启加载(load)。

流程：
1. 所有的写入命令会追加到aof_buf(缓冲区)中
2. AOF缓冲区根据相应的策略向盈满做同步操作
3. 随着AOF文件越来越大，需要定期对AOF文件进行重写，达到压缩的目的
4. 当Redis服务器重启时，可以加在AOF文件进行数据恢复

#### 命令写入
AOF命令写入的内容直接是文本协议格式。例如set hello world这条命令，在AOF缓冲区
会追加如下文本:<br>
*3\r\n$3\r\nset\r\n$5\r\nhello\r\n$5world\r\n  

#### 文件同步
Redis提供了多种AOF缓冲区同步文件策略，由参数appendfsync控制

配置值 | 说明
:---: | :---:
always | 命令写入aof_buf后调用系统fsync操作同步到AOF文件，fsync完成后线程返回
everysec | 命令写入aof_buf后调用系统write操作，write完成后线程返回。<br>fsync同步文件操作由专门的线程每秒调用一次
no | 命令写入aof_buf后调用系统write操作，不对AOF文件做fsync同步，<br>同步硬盘操作由操作系统负责，通常同步周期最长30秒

系统调用write和fsync说明
- write操作会触发延迟写(delayed write)机制。Linux在内核提供页缓冲区用来提高硬盘I/O性能。
  write操作在写入系统缓冲区后直接返回。同步硬盘操作依赖于系统调度机制，如：缓冲区页空间
  写满或达到特定的时间周期。同步文件之前如果此时系统故障宕机，缓冲区内数据将丢失
- fsync针对单个文件操作(比如AOF文件)，做强制硬盘同步，fsync将阻塞直到写入硬盘
  完成后返回，保证了数据持久化

说明
- 配置为always时，每次写入都要同步AOF文件，在一般的SATA硬盘上，Redis只能支持大约
  几百TPS写入，显然跟Redis高性能特性背道而驰，不建议使用
- 配置为no，由于操作系统每次同步AOF文件的周期不可控，而且会加大每次同步硬盘的
  数据量，虽然提升了性能，但数据安全性无法保证
- 配置为everysec，是建议的同步策略，也是默认配置，做到兼顾性能和数据安全性。
  理论上只有在系统突然宕机的情况下丢失1秒的数据

#### 重写机制
随着命令不断写入AOF，文件会越来越大，为了解决这个问题，Redis引入AOF重写机制
压缩文件体积。AOF文件重写是把Redis进程内的数据转化为写命令，同步到新的AOF文件
的过程。
<br>
重写后的AOF文件变小原因
1. 进程内已经超时的数据不再写入文件
2. 旧的AOF文件含有无效命令
3. 多条谢明令可以合并为一个

重写过程可以手动和自动触发
- 手动：直接调用bgrewriteaof
- 自动：根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage确定
  自动触发时机
  
#### 重启加载

1. AOF持久化开启且存在AOF文件时，优先加载AOF
2. AOF关闭或无AOF文件时，加载RDB
3. 加载AOF/RDB成功后，Redis启动成功
4. AOF/RDB文件存在错误时，Redis启动失败，并打印错误信息

#### 文件校验
加载损坏的AOF文件时会拒绝启动。<br>
AOF文件可能存在结尾不完整的情况，如机器突然掉电导致AOF尾部命令写入不全。
Redis提供了aof-load-truncated配置来兼容这种情况，默认开启。加载AOF时，
遇到此问题时会忽略并继续启动
