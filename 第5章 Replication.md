# 第5章 Replication

Replication 的意思是，把一份数据同时保存在多个机器上。使用 Replication 的原因有

- 让数据与终端用户更近，减小用户访问数据的延迟
- 提高可靠性，即使一部分机器崩溃了，数据依然可以访问
- 提高读取速度的吞吐量

在 Replication 的过程中，必须要保持各个 replication 的数据一致，目前主流的方法有

- single leader
- multi leader
- leaderless

### 数据同步

先说说什么是 leader。在前两种方法中，整套系统中至少会有一个 leader，不是 leader 的其他机器就是 follower 。leader 会接受用户的写请求，在写请求完成后，leader 将写入的数据发送给 follower，这样 leader 和 follower 的数据就一致了。读请求发送给 leader 或者 follower 都行

在上面的写请求处理过程中，有一个需要处理的问题，即 leader 什么时候告诉用户写请求完成了。这里有两种方式：一个是在 leader 把数据写入到本地后立即返回；第二个是在 leader 把数据都发送给所有的 follower 后返回。前者是**同步的（synchronously）**，后者是**异步的（asynchronously）**

同步的好处是，所有的 follower 的数据与 leader 是**一致的（consistency）**，但是缺点是用户需要等待很久才能得到返回

异步的好处是，leader 写完后就立即给用户返回，用户等待的时间很短，但是如果 follower 在接收到数据之前就崩溃了，那么 follower 的数据就与 leader 不一致

有一种结合了同步和异步的方法，叫做**半同步（semi-synchronously）**。这时系统里 leader 与某一个 follower 之间是同步的，与其他 follower 是异步的，这样至少能保证数据在一个 follower 上是一致的。如果这个 follower 崩溃了，那么就从余下的 follower 中选中一个做同步复制

在绝大多数的实际系统中，使用的方法都是异步的，虽然有数据丢失的风险，但是比起用户的超长等待时间，冒这个风险是值得的

### 新的 follower

系统运行过程中，有可能需要加入新的 follower，那么如何让新的 follower 的数据与 leader 一致呢？这其实也很简单

1. leader 可以定时保存自己当前的状态到一个 snapshot，这个 snapshot 可以用于备份等
2. 发送 snapshot 给新的 follower
3. 然后 follower 请求自从 snapshot 之后的所有变化，snapshot  在 leader 的数据中的位置被称为 log sequence number
4. 这时 follower 就已经与 leader 的数据同步了，接下来就可以照正常逻辑接受 leader 的数据变化的推送了

### 故障处理

系统中的任意节点都可能发生故障，在故障发生后，系统必须正常提供服务

follower 也和 leader 一样，在自己的磁盘上维护了一个快照（snapshot）。如果 follower 故障后，然后重启，可以自己的快照里快速恢复，然后向 leader 请求自从这个快照后的所有变化

如果 leader 崩溃了，这时需要发生一个叫做 **failover** 的操作：即一个 follower 要成为 leader，并对外提供服务。failover 可以由管理员手动出发，也可以自动触发。下面是自动出发的步骤

1. 监测 leader 崩溃了。监测的方式就是使用心跳，当 leader 超过一段时间（比如说 30 秒）不返回心跳结果后，既可认为 leader 崩溃了
2. 选择一个 follower 作为 leader。选择的方式很多，可以通过选举（election），或者预先由 controller node 指定一个 follower。总之，最佳的节点是拥有最新的数据的节点，这样能让数据丢失的风险降到最小
3. 重新配置系统。这时客户端需要将请求都发送给新的 leader。旧的 leader 如果重启了，必须要转变成 follower

failover 过程中可以会发生下面这些事情

1. 如果数据同步是异步的，那么那些还在旧的 leader 上的、没有同步给新的 leader 的数据怎么办？最简单的方法就是丢弃这些数据
2. discarding data is dangerous if other storage systems outside of the database need to be coordinated with the database contents。Github 就因此发生过一次事故
3. 避免裂脑（split brain）问题。裂脑指的是系统中出现了多个 leader，这时需要一个机制能检测到多个 leader、然后关闭直到只剩下一个 leader
4. 如何选择一个 leader 的 timeout 时间。太短了会导致很多不必要的 failover，这种情况会在网络较为拥塞时发生，非必要的 failover 又会加重网络拥塞；timeout 太长了又会在崩溃发生后丢失更多的数据

### 同步数据的方式

leader 和 follower 之间如何同步数据？下面介绍三种方法

1. 命令式（statement-based replication）
2. 日志式（WAL）
3. trigger

一种同步数据的方式是 **statement-based replication**。即 leader 将在他身上执行的命令，全都发送给 follower，让 follower 也执行一次。这个方式听起来很简单，但是也有一些需要注意的地方-

- 如何处理非确定函数的结果（nondeterministed function）。比如 MySQL 中的 `now()` 函数、 `rand()` 函数。一个可行的方式是，leader 同时发送这些非确定函数的结果给 follower，这样在 follower 里非确定函数的结果就都是确定得了
- statement 在 follower 必须以一致的顺序执行
- statement 可能会导致 side effect。必须要保证这些 side effect 的结果与 leader 上一直

 另外一种方式就是使用 write-head log（WAL）来同步数据。这里的日志（log）指的是以追加的方式添加到文件的一段数据（a sequence of byte）。我们在自谦的几个章节也聊到过日志

- SSTable 和 LSM-Tree 使用日志来保存数据
- B-Tree 使用日志，以在数据库崩溃后恢复数据

通过日志来同步数据，让 leader 不仅把日志写入自己的磁盘上，还发送给 follower，follower 就能从日志构建完整的数据

日志方式的缺点在于，日志的格式可能在不同的存储引擎、或者相同的存储引擎的不同版本中是不同的，这样日志就无法同步了

为解决上面的问题，我们可以让 leader 发送的日志与实际保存在磁盘上的日志不同，用于同步的日志就叫做逻辑日志（logical log）。MySQL 的 binlog 就是使用这种方法

上面的两个同步方式都是数据库自带的，二有时候我们希望在应用层来处理同步逻辑，这时候可以使用 trigger。trigger 是很多数据库自带的功能，其实就是一个事件监听器，应用程序注册了某些事件，当事件发生是，就能调用应用程序中的一个方法

这个方法的缺点是性能开销比较大，好处是非常灵活

### 数据同步的延迟

之前说过，使用很多的 replica 的好处是，读请求可以发送给 follower，这样就减小了 leader 的压力。为了提供更高的读吞吐量，我们可以增加更多的 follower。但是这样会引起一些问题

- 如果 leader 和 follower 之间的数据同步是同步的（好拗口），那么越多的 follower，意味着 leader 需要把数据发送给更多的 follower，一次写请求的耗时就更大。而且一旦其中一个 follower 崩溃了，会导致整个系统停止响应
- 如果数据同步是异步的，那么当你写入一个数据后，立即发送读请求，这些数据可能还没有同步到 follower 上，这意味着你的读请求可能不会返回你刚刚写入的数据，就好像数据消失了一样。这个问题叫做 eventual consistency 中的 read-your-write consistency

解决 read-your-write consistency 的方法有很多

- 对于可能被修改数据，从 leader 读取。比如说一个用户的账户信息，只可能被用户自己修改，所以读取自己的账户信息需要从 leader 读取，而读取其他人的账户信息可以从 follower 读取
- 如果上一次修改数据的时间在一分钟以内（这个时间需要根据实际情况选取），则从 leader 读取数据，否则从 follower 读取数据
- 书上还有一些方法我没看懂，我自己感觉跟上面这个方法差不多一样
- 如果需要监测不同节点的时间，那么可以不用记录真实的时间，而是记录逻辑时间（logical timestamp），比如说 log sequence number

如果用户会从多个设备访问应用的话，要注意在这些设备上都要实现 consistency

### 时光倒流

另一个反常现象是**时光倒流（moving back in time）**

这个现象发生的原因是有两个 follower，他们各自与 leader 的延迟不一样。比如一个用户向数据库插入了一条数据，然后 leader 把数据同步到 follower A 和 follower B，数据会先同步到 A，然后过一段时间才同步到 follower B。如果说在这之间，另一个用户先从 A 读取数据，他能获取到数据，但是当他第二次从 B 获取数据时，他发现这个数据又不存在了，仿佛自己穿越了

Monitonic  reads  会保证这样的错误不会发生。Monotinic reads 提供比 read-you-write consistency 更强的一致性

一个达到这样的一致性的方法，就是保证一个用户的读请求总是发送到同一个 follower。那么选择 follower 时可以用用户的用户名做哈希然后对 follower 的总数取余得到

### Consistent Prefix Reads

这个也是一种一致性要求，他保证如果一系列写操作以特定的顺序执行，name 任何一个人读取这些数据时，也会得到一致的顺序

解决这个问题的方法是，保证每条写入的数据之间没有太强的关联（causally related）
