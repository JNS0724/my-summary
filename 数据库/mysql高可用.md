# mysql 高可用

## 主从同步

mysql可以通过binlog来实现主从同步.

有异步/同步/半同步几种方式,默认情况下是异步

半同步是5.5之后才出现的,性能和可靠性上取得平衡.

半同步模式：主库会等待至少有一个从库把数据写入relay log并ACK完成，才成功返回结果。 半同步模式介于异步和全同步之间。

半同步的复制方案是在MySQL5.5开始引入的，普通的半同步复制方案步骤如下图：

* Master节点写数据到Binlog，并且执行Sync操作。
* Master发送数据给Slave节点，同时commit主库的事务。
* 收到ACK后Master节点把数据返回给客户端。

上面模式有点问题,主库先commit如果宕机会导致幻读.所以5.7改为先收到ack之后再commit主库事务.

## MHA模式

MySQL Master High Availability, 只负责mysql主节点的高可用,主库发生故障时,会从从节点中选择一个成为主节点.

1.即使是主从同步,也有数据丢失的可能.对于主从同步的数据丢失,可以用binlog server,模拟slave节点接收binlog日志记录,每次写入都要等binlog server的ack应答,故障时从binlog server获取最新数据.

2.mha管理节点有单点问题,相当于我们又要考虑mha节点的可靠性.因此可以通过在mysql主从之中引入agent,发生故障时每个Agent均参与选举投票，选举出合适的Slave作为新的主库，防止只通过Manager来切换，去除MHA单点。

## MGR

基于paxos或raft的分布式协议实现,当数据库发生故障时，MySQL内部自己进行切换。

相当于每次写入都要投票,半数以上ack了才写入,主节点挂掉之后由协议控制选主,把上面的两步都做到mysql的主从集群里.
