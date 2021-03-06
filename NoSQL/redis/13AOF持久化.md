# AOF持久化

## 保存数据

aof持久化保存数据库数据的方法是：每当有修改数据库的命令被执行时，服务器就会将执行的命令写入到aof文件的末尾。

## 数据还原

因为aof文件里面储存了服务器执行过的所有数据库修改命令，所以给定一个aof文件，服务器只要重新执行一遍aof文件里面包含的所有命令，就可以达到还原数据库的目的。

## 安全性问题

虽然服务器每执行一个修改数据库的命令，就会将被执行的命令写入到aof文件，但这并不意味着aof持久化不会丢失任何数据。在目前的操作系统中，执行系统调用write函数，将一些内容写入到某个文件里面时，为了提高效率，系统通常不会直接将内容写入到硬盘里面，而是先将内容放入到一个内存缓冲区里面。

因此，aof持久化在遭遇停机时丢失命令的数量，取决于命令写入到硬盘的时间：

- 越早将命令写入到硬盘，发生意外停机时丢失的数据就越少
- 而越迟将命令写入到硬盘，发生意外停机时丢失的数据就越多

## 安全性控制

为了控制redis服务器在遇到意外停机时丢失的数据量，redis为aof持久化提供了`appendfsync`选项。

always：服务器每写入一个命令，就调用一次fdatasync，将缓冲区里面的命令写入到硬盘里面

everysec：服务器每秒钟调用一次fdatasync，将缓冲区里面的命令写入到硬盘里面。在这种模式下，服务器遭遇意外停机时，最多丢失一秒钟内执行的命令数据

no：服务器不主动调用fdatasync，由操作系统决定何时将缓冲区里面的命令写入到硬盘里面，在这种模式下，服务器遭遇意外停机时，丢失命令的数量是不确定的。

运行速度：always速度慢，everysec和no都很快

默认值：everysec

## aof重写

随着服务器的不断运行，为了记录数据库发生的变化，服务器将越来越多的命令写入到aof文件里面，是的aof文件的体积不断地增大。

为了让aof文件的大小控制在合理的范围，避免它胡乱的增长，redis提供了aof重写功能，通过这个功能，服务器可以产生一个新的aof文件：

- 新aof文件记录的数据库数据与原有aof文件记录的数据库数据完全一样
- 新的aof文件会使用尽可能少的命令来记录数据库数据，因此新aof文件的体积通常比原有aof文件的体积要小的多
- aof重写期间，服务器不会被阻塞，可以正常处理客户端发送的命令请求。

### aof重写的出发

1. 客户端向服务器发送BGREWRITEAOF命令
2. 通过设置配置选项来让服务器自动执行BGREWRITEAOF命令
   1. auto-aof-rewrite-min-size <size> 出发aof重写所需的最小体积：只有在aof文件的体积大于size时，服务器才会考虑是否需要进行aof重写，这个选项用于避免体积过小的aof文件进行重写
   2. auto-aof-rewrite-percentage <percent> 指定触发重写所需的aof文件体积百分比：当aof文件的体积大于auto-aof-rewrite-min-size指定的体积，并且超过上一次重写之后的aof文件体积的percent%时，就会触发aof重写。如果服务器刚刚启动不久，还没有进行aof重写，那么使用服务器启动时载入的aof文件的体积来作为基准值。将这个值设置为0表示关闭自动aof重写。

## 持久化对比

| rdb                                                          | aof                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 全量备份，一次保存整个数据库                                 | 增量备份，一次保存一个修改数据库的命令                       |
| 保存时间间隔较长                                             | 保存的间隔默认为一秒钟                                       |
| 数据还原速度快                                               | 数据还原速度一般。冗余命令越多，还原速度越慢                 |
| 执行save命令时会阻塞服务器，但手动或者自动触发的bgsave都不会阻塞服务器 | 无论是平时还是进行aof重写时，都不会阻塞服务器                |
| 更适合数据备份                                               | 更适合用来保存数据，通常意义上的持久化。在appendfsync always模式下运行时，redis的持久化方式和一般的sql数据库持久化方式是一样的。 |

可以同时使用两种持久化，根据你的需求来判断，还原数据库优先使用aof文件。