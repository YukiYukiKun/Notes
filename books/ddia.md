# 可靠性 可拓展性 可维护性

数据系统是什么
===
DB MQ Cache 都可以属于数据系统。DB 和 MQ 的区别正越来越模糊。
目前流行把系统中的各个部分抽出并分别实现。例如缓存(redis) 搜索(es) DB 三者拆分，在应用层设法将其结合。

我们的目标
===
数据系统最重要的问题有以下三点
- 可靠性 就是能扛住（集群的）部分故障。
- 可拓展性 就是可以方便的扩展，扛住增长的负载。
    1. 文中举出了推特的例子：读扩散和写扩散的权衡。一开始推特使用读扩散，结果发现扛不住。后来换成了写扩散。但是这样的话，大V发推时将会带来沉重负担。目前推特尝试使用混合方式，即大V不搞写扩散，而是用读扩散。其他普通用户用写扩散。用户读推时再做请求和合并。
    2. 描述性能时，我们以响应时间和吞吐量为标准进行衡量。
    3. 垂直扩展：增加单点能力；水平扩展：增加节点数目。
    4. 考虑扩展，需要对系统未来的负载做出假设和预估。
- 可维护性 好用
    1. 易于运维。例如兼容性 滚动升级等。
    2. 方便改。良好的抽象易于扩展。

# 数据模型 & 查询语言

关系模型 & 文档模型
===

两种模型
---
- 关系模型  数据被组织成关系（即表）
- 文档模型  即 NoSQL。它为了实现（分布式）可拓展性而做出了一些妥协，如不能完成关系数据库的一些特殊查询操作，或反范式化。

对象关系不匹配
---
对于传统关系数据库，程序和DB中的数据组织方式是不一样的，这带来了额外的互相转换的工作。ORM可以用来处理这种转换。NoSQL可以更好的解决这个问题。
举例来说，传统的SQL需要用外键表示嵌套结构，而新的NoSQL数据库支持直接对一坨JSON对象进行存储和查询，更加直观也一致。

关系分类
---
- 一对一  最简单的kv
- 一对多  也是很简单的情况，可以用文档数据库不分表。
- 多对一 & 多对多  都是较为复杂的情况，需要分表来避免冗余（保持范式）。也是关系数据库擅长的。

其中，前两者很适合用单个层次结构/文档模型来表示。而为了针对后两种，人们发明了关系数据库和树模型。**文档数据库反而有返璞归真的感觉。**

文档和关系哪个更方便
---
文档模型在关系不复杂，嵌套不深时，使用起来更为方便。
但是如果有大量多对多多对一关系，文档数据库只能通过文档引用而不能在一次查询中完成，或直接通过反范式完成。
所以多对多和高度相联的数据，是可以用关系数据库的。反之，如果大部分都是一对一一对多的简单情况（比如游戏），则可以用文档数据库。

数据模式
---
模式的实现分为写时模式（有模式）和读时模式（无模式）的做法。后者更加的放飞自我。

# 存储与检索

数据库的数据结构
===

原始结构
---
一个文件，增和改的记录都追加在最后面。查询的时候找到最后面一条符合要求的记录即可。
append-only 的结构极其高效和常用。
这种原始结构查询是 O(n)，所以需要引入索引。

Hash 索引
---
在上述结构的基础上，在Hash表中为每个 key 维护一个数据文件中的位置。

数据文件分块
---
单个数据文件达到一定长度后，可以停止写入并开一个新文件。
完成写入的文件可以进行整理（去除单个文件中的重复老数据）和合并（合并多个数据文件）。这就是 ES 中 Segment 处理的思路。
每个文件依然有独立的索引。查找时从新到旧查找。

原始结构的优点
---
- 没有随机写，对磁盘很友好。
- 天然不怕崩溃，因为不会修改旧数据。
- 虽然写只能单线程，但读可以并发。

Sorted String Tables (SSTables)
===
在前述简单结构的基础上，保证每块结构中 key 按照顺序排列。

好处
---
由于有顺序，合并时非常顺畅，而且查找时将不再需要额外的索引。

实现
---
在内存中维护有序结构（binary-search-tree 甚至 B~tree），尺寸达到阈值后则写入磁盘，成为一个 Segment。查找时，从新到旧搜索文件。
后续也会进行 Segment 合并。
为了处理崩溃问题，还将引入顺序写入的 redo 日志。
**这和典型的 DB 已经非常相似**

Log-Structured Merge Tree (LSM tree)
===
利用上述带有日志的 SSTable 结构构建出的就是 LSM tree 。Lucene (和 ES) 就是用这种结构组织索引。每个倒排索引就是上述的一个结构，而单个 term 就是一个 key。

LMS tree 优化
---
上面的查找方式使得查找不存在的条目时会很慢。引入 Bloom Filter 可以解决这一问题。

B tree
===
这里我们再次强调，B树是一种在磁盘中的数据组织形式，而（通常）不（只）是索引。因为仅仅是索引的话，完全可以全放进内存。
B树的思路仍然是排序的kv结构。它考虑了磁盘的组织方式。与前述 SSTables 分块相对应，B 树以（通常为4kB）的小块为树的节点。这也是为了迁就磁盘的结构。
**分支因子**：即为分叉数。通常为几百。这使得大部分数据库都可以放在三到四层的树中。
**redo log**：B tree 这种涉及修改的结构，必须引入 立即刷入磁盘的 append-only 日志。通常使用典型的 redo log。

B tree 优化
---
- 对节点的修改使用 Copy-on-Write ，可以简化崩溃恢复。
- 放置叶子节点时尽量使其在磁盘上连续。不过比起 LMS tree，B tree 要做到这一点不容易。
- 兄弟结点之间增加指针，可以加快顺序扫描。
- 分形树。

B tree 和 LSM 树的比较
---
这两种结构分别代表了 DB 的两种组织形式：就地修改结构 或 基于日志。

总体来说 LSM tree 写入占优，而 B tree 读取占优。
**LSM tree 优势**
- LSM tree 的写放大（也就是写入的额外操作）更少。因为 B tree 必须写入 redo log 和树本身。而且 B 树通常需要修改块的大部分。
- LSM tree 压缩后的结构更加紧凑。
**LSM tree 劣势**
- 进行压缩时，磁盘 IO 非常集中，很可能影响其他写入操作。
- 需要小心配置，使得压缩的频率能够跟上写入的速度，同时又不会因为太频繁而影响正常写入。
- LSM tree 结构中存在 key 的副本。而 B tree 永远只有一份，在事务中对 key 加锁更加方便。

其他索引结构
---
上面的 LSM tree 和 B tree 讨论的都是主键索引（聚簇索引）。不过它们也可以用作二级索引。二级索引需要考虑 key 重复的问题，不过比较 trivial。

多键索引
---
- 直接拼接的复合索引
- 多维（如地理经纬度）索引：使用空间填充曲线或使用 R 树。

OLTP/OLAP
===
日常用的 OLTP 和分析用的 OLAP 需求上有所不同。OLTP 需要注重并发读写事务，更经常取出若干整条数据；而 OLAP 的数据是集中导入的，它对大规模数据进行统计和组织，通常针对部分列。
**雪花式分析**：以一个事实表的信息为材料（如交易记录），建立出其他统计的表（如商家、商品统计）。

列式存储
---
由于 OLAP 通常访问大量的行中的部分列且不经常查询个别的行，它更适合使用列式存储。所谓的列式存储就是把每列存放在一起。所有列中数据的顺序需要一致，以此我们才能定位东西在哪一行。因此我们也不能只对个别列排序，要排就得一起排。

需要留意的是，OLTP 系统需要更加留意磁盘、缓存的带宽吞吐量。为此可以引入列压缩，因为有的列值是离散和重复的。

# 编码和演化

需要留意的是，编码中是否带有元数据，是否可以自解释。
- JSON XML 这种可以自解释的，它们更臃肿。
- Thrift 和 protobuf 这种需要额外元数据才能解释的，但它们带有额外的帮助解释的信息（字段序号），演化性/兼容性更好。
- 咱项目里最紧凑的二进制形式，不包含任何元数据。紧凑的代价是几乎没有兼容性。对于不持久的数据才可以这样。

分布式系统的演化问题
---
分布式系统的演化问题，还需要考虑对滚动升级的支持。不同节点版本不同时，需要考虑前向后向兼容。

这一章大部分内容咱跳过 (｀・ω・´)

# 复制
复制带来以下好处：
- 更健壮
- 分散读负载
- 为地理上不同地方的用户提供服务

复制有以下形式：
- 单主
- 多主
- 无主

Leader & Follower
===
只有 Leader 才能接受写入请求。Follower 从 Leader 处复制数据。

同步和异步复制
---
**同步复制**：Leader 修改请求必须等到从库修改成功。
**异步复制**：修改直接成功，指不定什么时候同给从库。
这两种从库可以混用，即**半同步**。做法是指定一个从库进行同步复制，其他的随缘。
大部分情况使用纯异步。

存在同步复制时，系统对持久化和健壮性有更强保证。纯异步牺牲了这点，不过它是最广泛的方案。

新增 Follower
---
初始化新的 Follower 需要以下步骤：
1. 复制 Leader 某一时刻的快照（RDB之类的）
2. 从快照时刻之后利用 Leader 的 redo log 来追上leader，则达成一致。

从库失效：追赶恢复
---
用 redo log 追就好，trivial。

主库失效：故障切换
---
通常为以下步骤：
1. 确认 Leader 真的失效
2. 选举一个 leader。尽量找一个版本号最新的，可能需要用到共识算法。
3. 更新配置。写请求发向新 leader，且其他节点都应认可它，包括之前的旧 leader。

可能出现的问题，也是分布式理论的经典问题。理论上没有完美解法：
- 异步情况下，新 Leader 落后于旧 Leader，持久性没能保证。
- 如果领先的旧 Leader 又回来了，对于一些版本号的修改就产生了冲突。此时可以选择无视旧 Leader 的数据。
- 如果集群和其他数据系统有交互，那么丢弃旧 Leader 数据更是危险。如，故障切换的 redis 集群忘记了过去产生的主键，自动生成时和过去的主键产生了重复。而系统可能已经拿这些主键在 Mysql 里创建了条目。如此就发生了信息的泄露。**设法维护发号器等来解决？**
- 脑裂。可以试图关闭其中一个（fencing），也可直接设置集群生效数量。
- Leader 失效判定的权衡：严格还是宽松？

复制日志的实现
===

基于语句的复制
---
即直接转发语句，在 follower 上重放。但会有以下问题：
- 带有非确定函数的语句将会造成不一致。
- 考虑指令乱序情况，和当前 DB 状态有关的语句将会造成不一致（如带 where 的）。
- 有（用户定义的）副作用函数都会造成不一致。
该方式限制太多，使用的情况较少

基于WAL（redo log）的复制
---
传输 LSM tree 里的日志，或其他（如 B tree 里的） redo log。统称为 **Write Ahead Log** 更为合理，总之是前面数据库架构里提到的那种 append only 日志。这种日志相对底层（通常是以磁盘哪块哪行做了哪种修改的形式），且需要考虑前后兼容。

逻辑日志复制（基于行）
---
比起上面的日志更加抽象，通常是行添加、修改的内容。涉及多行的事务将会产生多行行日志。逻辑日志兼容性和可读性更强。

复制延迟问题
===
大集群更不可能用同步复制的方式。但异步复制必有延迟。
对于这个问题，现在流行所谓的“最终一致性”，也就是认为集群收敛是异步的，且对时间没有保证。
下面讨论发生复制延迟时**以用户为角度**出现的问题。

Read Your Writes 读所写/写后读
---
某请求向 leader 写入，随后试图读取，却被路由到了复制落后的 follower，产生了明显的不一致。
为避免此种情况，我们需要保证 **read-after-write consistency**。有如下解法：
- 所有可能被用户编辑的内容，都从 Leader 读。例如用户只能编辑自己的个人信息。若用户能改的东西太多，这种方法就不能用了。
- 在进行修改之后的一段时间，都从主库读。
- 记录最后一次修改的版本号/(逻辑)时间戳，只从比该时间戳新的 follower 读。 **考虑这里如何路由，或许可以在集群管理信息中维护所有节点的当前版本**；另外，**维护逻辑时间戳时需要考虑 follower 读取乱序的可能。**

Monotonic Reads 单调读
---
使用集群时请求节点发生改变，可能会突然看到更旧的东西。故需要确保用户后读到的信息一定比之前读到的新。
这里只提到一个解法：设法确保单个用户（短时间内）只从固定 follower 读。

Consistent Prefix Read 一致前缀读
---
Follower 和 leader 的写入顺序不同造成的混乱。可能客户端会先看到在 leader 逻辑上后写入的信息。
为保证一致前缀读，我们需要维护写入之间的因果逻辑关系 （Lamport Timestamp 及其变体），依此保证其写入顺序（后面的章节会讨论）。

复制延迟解决方案总览
---
在大型分布式系统中，放弃 ACID 事务退而求其次使用最终一致性保证是合理的。根据业务的特点和容忍性，对于上述问题可以提供不同程度的保证。

多主复制
===
这里指的不是 sharding（它理论上仍是单主），而是在所有 leader 都可以修改所有数据的做法。进行修改后，所有 leader 互相复制。多主复制复杂且危险，**本书明确指出应避免多主**。

写入冲突
---
俩 leader 同时对一处数据进行修改，复制时就爆炸。而且冲突没有“正确的”解决方式。
有以下方法处理这种情况：
- Sharding：
    就是单主的效果。但要考虑故障切换时如何保证正确（参考 Redis 集群）。
- Last Write Win (LWW) 最后写入胜利：
    设法为每次写入分配有全集群唯一的有全序关系的Id（如 雪花 snowflake），冲突时更新的赢。
- 节点优先级：
    为每个节点分配优先级或唯一Id，冲突时大的赢。
- 自动冲突解决
    写点什么自定义逻辑把两个写入结合起来。可以在写或读的时候处理。
- 保留冲突
    保留冲突，日后留待人工决定。

对于自动冲突解决，有以下新技术可以留意：
- 无冲突复制数据类型（Conflict-free replicated datatypes）（CRDT）
    某些数据结构可以直接合并，如有序列表，计数器，集合等。
- 可合并的持久数据结构（Mergeable persistent data structures）
    带版本控制的数据结构，用三向合并解决冲突。
- 可执行的转换（operational	transformation
    太简略没看懂。

多主复制拓扑
---
多个 leader 可以以以下拓扑结构组织：
- 圈圈
- 星形
- 全连接
前两者有 leader 缺失便会阻断拓扑，而全连接的在多主写入的情况下更容易出现乱序。需要使用 version vectors 对多条更新进行逻辑上的因果排序。

无主复制
===
很 fancy 就是了，这辈子都不敢碰。代表是 Dynamo。
基本思想是，客户端的读写请求都被同时发送给**所有**节点。

故障处理
---
当发生节点宕机时，不需要进行故障切换。只要有足够的节点存活，依然可以照常进行读写操作。即使有数据落后的节点回到集群，由于读操作也向多个节点请求，故亦不受影响。

读修复（Read repair）和反熵过程（anti-entropy process）
---
在上述同时写入的基础上，（尤其是宕机节点回归时）保持节点数据一致的重要机制。

- **读修复**：
    客户端驱动。当发现读取的多个节点数据中有落后的，客户端将把新值写入该节点。适合频繁读取的值。
- **反熵过程**
    由后台进程检查并主持修复节点间的数据不一致。反熵进行的复制流程相比前述主从复制延迟更大，（单调 一致前缀等）保证更少。它可以弥补读修复时冷 key 无法被修复的情况。

读写法定人数
---
总共 $n$ 个节点的系统中，写入需要有 $w$ 个节点成功才视为成功；读取则从 $r$ 个节点的结果中取得最新的（**但是我咋知道哪个是最新的...这个后面章节应该会讨论**）。这两个值应配合设置，以保证读取时可以得到来自至少一个节点的最新数据 $(w + r > n)$。上述条件成为法定条件。
这里，$w$ 和 $r$ 比 $n$ 小越多，我们就可以分别在读和写时容忍越多的节点失效。**注意这里容忍能力是读写分开的**，所以我们还可以根据需要调整读和写的容忍度（如 $w$ 为 1，$r$ 为 $n$ 这种极端情况）。
与前述对应，也可以不满足法定条件，即 $(w + r <= n)$。如此做牺牲了一致性保证而提高了效率和可用性。

读到陈旧数据的场景
---
即使有法定读写的保证，现实中还是可能会出现问题。

- 同时写入：
    和多主类似，由于是多点，我们无法判断同时发生的写入谁在前面。此时采用和多主一样的合并策略。
- 部分新值部分旧值：
    读取不同节点返回不同值，无法判断新旧。
- 写入失败后读取：
    当有成功的节点过少时，写入请求整体失败。但成功的节点若不进行回滚，后续的请求将会读到这次失败写入的值。故应设法回滚。
- 节点恢复
    数据较新的节点失效后，可能会从其他老节点获得恢复的数据，使集群中有老数据的节点增加。

陈旧度监控
---
有主的复制可以通过监控从节点的版本号来评估陈旧度，但无主很困难。

主从
---
无主的节点也可以引入接受复制的从节点。这些从节点只是在接受复制，而不算在 N 个节点中。

冲突问题
---
不同的节点接受写入请求时无法保证顺序，如果简单覆写，可能会导致各个节点最终状态不一致。

- 最后写入胜利：
    以“某种时间戳”确定写入的先后，并发写入时丢弃更老的写入。
    如此做会让一些写入被默默丢弃。所以会破坏持久化保证。
- happens-before 关系：
    有明确现后关系的只有有因果依赖的操作。而这种有因果依赖的操作是由明确先后关系，可以安全覆盖的。
    这里我们使用 **Lamport Timestamp** 或 **Version Vector**。注意，后者是每个节点维护了一个版本，各节点同时传递所有版本，但只有本机版本会+1。向量中所有版本都大于等于时确定是在后；小于等于则在前；其他情况都是不确定。

实践时，我们可以覆盖有确定因果关系的写入，并同时保留其他并列的写入。