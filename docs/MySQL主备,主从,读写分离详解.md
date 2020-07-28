# MySQL主备、主从、读写分离详解

# 一、MySQL主备的基本原理
![Nf2ZKU.png](https://s1.ax1x.com/2020/06/29/Nf2ZKU.png)

在状态1中，客户端的读写都直接访问节点A，而节点B是A的备库，只是将A的更新都同步过来，到本地执行。这样可以保持节点B和A的数据是相同的。当需要切换的时候，就切成状态2。这时候客户端读写访问的都是节点B，而节点A是B的备库

在状态1中，虽然节点B没有被直接访问，但是建议把备库节点B，设置成只读模式。有以下几个原因：

1.有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作

2.防止切换逻辑有bug

3.可以用readonly状态，来判断节点的角色

把备库设置成只读，还怎么跟主库保持同步更新？

readonly设置对超级权限用户是无效的，而用于同步更新的线程，就拥有超级权限

下图是一个update语句在节点A执行，然后同步到节点B的完整流程图：

![Nf2M5R.png](https://s1.ax1x.com/2020/06/29/Nf2M5R.png)

备库B和主库A之间维持了一个长连接。主库A内部有一个线程，专门用于服务备库B的这个长连接。一个事务日志同步的完整过程如下：

1.在备库B上通过change master命令，设置主库A的IP、端口、用户名、密码，以及要从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量

2.在备库B上执行start slave命令，这时备库会启动两个线程，就是图中的io_thread和sql_thread。其中io_thread负责与主库建立连接

3.主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发给B

4.备库B拿到binlog后，写到本地文件，称为中转日志

5.sql_thread读取中转日志，解析出日志里的命令，并执行

由于多线程复制方案的引入，sql_thread演化成了多个线程

# 二、循环复制问题
双M结构：

![Nf2UVH.png](https://s1.ax1x.com/2020/06/29/Nf2UVH.png)
节点A和节点B互为主备关系。这样在切换的时候就不用再修改主备关系

双M结构有一个问题要解决，业务逻辑在节点A上更新了一条语句，然后再把生成的binlog发给节点B，节点B执行完这条更新语句后也会生成binlog。那么，如果节点A同时是节点B的备库，相当于又把节点B新生成的binlog拿过来执行了一次，然后节点A和B间，会不断地循环执行这个更新语句，也就是循环复制

MySQL在binlog中记录了这个命令第一次执行时所在实例的server id。因此，可以用下面的逻辑，来解决两个节点间的循环复制问题：

1.规定两个库的server id必须不同，如果相同，则它们之间不能设定为主备关系

2.一个备库接到binlog并在重放的过程中，生成与原binlog的server id相同的新的binlog

3.每个库在收到从自己的主库发过来的日志后，先判断server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志

双M结构日志的执行流如下：

1.从节点A更新的事务，binlog里面记的都是A的server id

2.传到节点B执行一次以后，节点B生成的binlog的server id也是A的server id

3.再传回给节点A，A判断这个server id与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了
# 三、主备延迟

![Nf2rxf.png](https://s1.ax1x.com/2020/06/29/Nf2rxf.png)
## 1、什么是主备延迟？
与数据同步有关的时间点主要包括以下三个：

1.主库A执行完成一个事务，写入binlog，这个时刻记为T1

2.之后传给备库B，备库B接收完这个binlog的时刻记为T2

3.备库B执行完这个事务，把这个时刻记为T3

所谓主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值，也就是T3-T1

可以在备库上执行show slave status命令，它的返回结果里面会显示seconds_behind_master，用于表示当前备库延迟了多少秒

seconds_behind_master的计算方法是这样的：

1.每个事务的binlog里面都有一个时间字段，用于记录主库上写入的时间

2.备库取出当前正在执行的事务的时间字段的值，计算它与当前系统时间的差值，得到seconds_behind_master

如果主备库机器的系统时间设置不一致，不会导致主备延迟的值不准。备库连接到主库的时候，会通过SELECTUNIX_TIMESTAMP()函数来获得当前主库的系统时间。如果这时候发现主库的系统时间与自己不一致，备库在执行seconds_behind_master计算的时候会自动扣掉这个差值

网络正常情况下，主备延迟的主要来源是备库接收完binlog和执行完这个事务之间的时间差

主备延迟最直接的表现是，备库消费中转日志的速度，比主库生产binlog的速度要慢

## 2、主备延迟的原来
1. 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差

2. 备库的压力大。主库提供写能力，备库提供一些读能力。忽略了备库的压力控制，导致备库上的查询耗费了大量的CPU资源，影响了同步速度，造成主备延迟

可以做以下处理：

	- 一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力
	- 通过binlog输出到外部系统，比如Hadoop这类系统，让外部系统提供统计类查询的能力

3.. 大事务。因为主库上必须等事务执行完才会写入binlog，再传给备库。所以，如果一个主库上的语句执行10分钟，那这个事务很可能会导致从库延迟10分钟

典型的大事务场景：一次性地用delete语句删除太多数据和大表的DDL

# 四、主备切换策略
## 1、可靠性优先策略

双M结构下，从状态1到状态2切换的详细过程如下：

1. 判断备库B现在的seconds_behind_master，如果小于某个值继续下一步，否则持续重试这一步

2. 把主库A改成只读状态，即把readonly设置为true

3. 判断备库B的seconds_behind_master的值，直到这个值变成0为止

4. 把备库B改成可读写状态，也就是把readonly设置为false

5. 把业务请求切到备库B

![Nf25R0.png](https://s1.ax1x.com/2020/06/29/Nf25R0.png)
这个切换流程中是有不可用的时间的。在步骤2之后，主库A和备库B都处于readonly状态，也就是说这时系统处于不可写状态，直到步骤5完成后才能恢复。在这个不可用状态中，比较耗时的是步骤3，可能需要耗费好几秒的时间。也是为什么需要在步骤1先做判断，确保seconds_behind_master的值足够小

系统的不可用时间是由这个数据可靠性优先的策略决定的

## 2、可用性优先策略
可用性优先策略：如果强行把可靠性优先策略的步骤4、5调整到最开始执行，也就是说不等主备数据同步，直接把连接切到备库B，并且让备库B可以读写，那么系统几乎没有不可用时间。这个切换流程的代价，就是可能出现数据不一致的情况

	mysql> CREATE TABLE `t` (
	  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
	  `c` int(11) unsigned DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB;
	
	insert into t(c) values(1),(2),(3);
表t定义了一个自增主键id，初始化数据后，主库和备库上都是3行数据。继续在表t上执行两条插入语句的命令，依次是：

	insert into t(c) values(4);
	insert into t(c) values(5);

假设，现在主库上其他的数据表有大量的更新，导致主备延迟达到5秒。在插入一条c=4的语句后，发起了主备切换

下图是可用性优先策略，且binlog_format=mixed时的切换流程和数据结果
![NfRiee.png](https://s1.ax1x.com/2020/06/29/NfRiee.png)

1. 步骤2中，主库A执行完insert语句，插入了一行数据(4,4)，之后开始进行主备切换

2. 步骤3中，由于主备之间有5秒的延迟，所以备库B还没来得及应用插入c=4这个中转日志，就开始接收客户端插入c=5的命令

3. 步骤4中，备库B插入了一行数据(4,5)，并且把这个binlog发给主库A

4. 步骤5中，备库B执行插入c=4这个中转日志，插入了一行数据(5,4)。而直接在备库B执行的插入c=5这个语句，传到主库A，就插入了一行新数据(5,5)

最后的结果就是，主库A和备库B上出现了两行不一致的数据

可用性优先策略，设置binlog_format=row

![NfRUyT.png](https://s1.ax1x.com/2020/06/29/NfRUyT.png)

因此row格式在记录binlog的时候，会记录新插入的行的所有字段值，所以最后只会有一行不一致。而且，两边的主备同步的应用线程会报错duplicate key error并停止。也就是说，这种情况下，备库B的(5,4)和主库A的(5,5)这两行数据都不会被对方执行

## 3、小结
1. 使用row格式的binlog时，数据不一致问题更容易被发现。而使用mixed或者statement格式的binlog时，可能过了很久才发现数据不一致的问题

2. 主备切换的可用性优先策略会导致数据不一致。因此，大多数情况下，建议采用可靠性优先策略

# 五、MySQL的并行复制策略

![NfRyf1.png](https://s1.ax1x.com/2020/06/29/NfRyf1.png)

主备的并行复制能力，要关注的就是上图中黑色的两个箭头。一个代表客户端写入主库，另一个代表备库上sql_thread执行中转日志

在MySQL5.6版本之前，MySQL只支持单线程复制，由此在主库并发高、TPS高时就会出现严重的主备延迟问题

多线程复制机制都是把只有一个线程的sql_thread拆成多个线程，都符合下面这个模型：
![NfRotA.png](https://s1.ax1x.com/2020/06/29/NfRotA.png)

coordinator就是原来的sql_thread，不过现在它不再直接更新数据了，只负责读取中转日志和分发事务。真正更新日志的，变成了worker线程。而worker线程的个数就是由参数slave_parallel_workers决定的

coordinator在分发的时候，需要满足以下两个基本要求：

- 不能造成更新覆盖。这就要求更新同一行的两个事务，必须被分发到同一个worker中
- 同一个事务不能被拆开，必须放到同一个worker中

## 1、MySQL5.6版本的并行复制策略
MySQL5.6版本支持了并行复制，只是支持的粒度是按库并行。用于决定分发策略的hash表里，key是数据库名

这个策略的并行效果取决于压力模型。如果在主库上有多个DB，并且各个DB的压力均衡，使用这个策略的效果会很好

这个策略的两个优势：

- 构造hash值的时候很快，只需要库名
- 不要求binlog的格式，因为statement格式的binlog也可以很容易拿到库名

可以创建不同的DB，把相同热度的表均匀分到这些不同的DB中，强行使用这个策略

## 2、MariaDB的并行复制策略
redo log组提交优化，而MariaDB的并行复制策略利用的就是这个特性：

- 能够在同一个组里提交的事务，一定不会修改同一行
- 主库上可以并行执行的事务，备库上也一定是可以并行执行的

在实现上，MariaDB是这么做的：

1. 在一组里面一起提交的事务，有一个相同的commit_id，下一组就是commit_id+1

2. commit_id直接写到binlog里面

3. 传到备库应用的时候，相同commit_id的事务分发到多个worker执行

4. 这一组全部执行完成后，coordinator再去取下一批

下图中假设三组事务在主库的执行情况，trx1、trx2和trx3提交的时候，trx4、trx5和trx6是在执行的。这样，在第一组事务提交完成的时候，下一组事务很快就会进入commit状态
![NfWQc6.png](https://s1.ax1x.com/2020/06/29/NfWQc6.png)
按照MariaDB的并行复制策略，备库上的执行效果如下图：
![NfWtNd.png](https://s1.ax1x.com/2020/06/29/NfWtNd.png)
在备库上执行的时候，要等第一组事务完全执行完成后，第二组事务才能开始执行，这样系统的吞吐量就不够

另外，这个方案容易被大事务拖后腿。假设trx2是一个超大事务，那么在备库应用的时候，trx1和trx3执行完成后，下一组才能开始执行。只有一个worker线程在工作，是对资源的浪费


## 3、MySQL5.7版本的并行复制策略
MySQL5.7版本由参数slave-parallel-type来控制并行复制策略：

- 配置为DATABASE，表示使用MySQL5.6版本的按库并行策略
- 配置为LOGICAL_CLOCK，表示的就是类似MariaDB的策略。MySQL在此基础上做了优化

同时处于执行状态的所有事务，是不是可以并行？

不可以，因为这里面可能有由于锁冲突而处于锁等待状态的事务。如果这些事务在备库上被分配到不同的worker，就会出现备库跟主库不一致的情况

而MariaDB这个策略的核心是所有处于commit状态的事务可以并行。事务处于commit状态表示已经通过了锁冲突的检验了
![NfWW3q.png](https://s1.ax1x.com/2020/06/29/NfWW3q.png)
其实只要能够达到redo log prepare阶段就表示事务已经通过锁冲突的检验了

因此，MySQL5.7并行复制策略的思想是：

1. 同时处于prepare状态的事务，在备库执行时是可以并行的

2. 处于prepare状态的事务，与处于commit状态的事务之间，在备库执行时也是可以并行的

binlog组提交的时候有两个参数：

- binlog_group_commit_sync_delay参数表示延迟多少微妙后才调用fsync
- binlog_group_commit_sync_no_delay_count参数表示基类多少次以后才调用fsync

这两个参数是用于故意拉长binlog从write到fsync的时间，以此减少binlog的写盘次数。在MySQL5.7的并行复制策略里，它们可以用来制造更多的同时处于prepare阶段的事务。这样就增加了备库复制的并行度。也就是说，这两个参数既可以故意让主库提交得慢些，又可以让备库执行得快些

## 4、MySQL5.7.22的并行复制策略
MySQL5.7.22增加了一个新的并行复制策略，基于WRITESET的并行复制，新增了一个参数binlog-transaction-dependency-tracking用来控制是否启用这个新策略。这个参数的可选值有以下三种：

- COMMIT_ORDER，根据同时进入prepare和commit来判断是否可以并行的策略
- WRITESET，表示的是对于事务涉及更新的每一行，计算出这一行的hash值，组成集合writeset。如果两个事务没有操作相同的行，也就是说它们的writeset没有交集，就可以并行
- WRITESET_SESSION，是在WRITESET的基础上多了一个约束，即在主库上同一个线程先后执行的两个事务，在备库执行的时候，要保证相同的先后顺序

为了唯一标识，hash值是通过库名+表名+索引名+值计算出来的。如果一个表上除了有主键索引外，还有其他唯一索引，那么对于每个唯一索引，insert语句对应的writeset就要多增加一个hash值

1. writeset是在主库生成后直接写入到binlog里面的，这样在备库执行的时候不需要解析binlog内容

2. 不需要把整个事务的binlog都扫一遍才能决定分发到哪个worker，更省内存

3. 由于备库的分发策略不依赖于binlog内容，索引binlog是statement格式也是可以的

对于表上没主键和外键约束的场景，WRITESET策略也是没法并行的，会暂时退化为单线程模型

# 六、主库出问题了，从库怎么办？
下图是一个基本的一主多从结构
![NffJMV.png](https://s1.ax1x.com/2020/06/29/NffJMV.png)
图中，虚线箭头表示的是主备关系，也就是A和A’互为主备，从库B、C、D指向的是主库A。一主多从的设置，一般用于读写分离，主库负责所有的写入和一部分读，其他的读请求则由从库分担
![NffcqO.png](https://s1.ax1x.com/2020/06/29/NffcqO.png)
一主多从结构在切换完成后，A’会成为新的主库，从库B、C、D也要改接到A’

## 1、基于位点的主备切换
当我们把节点B设置成节点A’的从库的时候，需要执行一条change master命令：

	CHANGE MASTER TO 
	MASTER_HOST=$host_name 
	MASTER_PORT=$port 
	MASTER_USER=$user_name 
	MASTER_PASSWORD=$password 
	MASTER_LOG_FILE=$master_log_name 
	MASTER_LOG_POS=$master_log_pos  

- MASTER_HOST、MASTER_PORT、MASTER_USER和MASTER_PASSWORD四个参数，分别代表了主库A’的IP、端口、用户名和密码
- 最后两个参数MASTER_LOG_FILE和MASTER_LOG_POS表示，要从主库的master_log_name文件的master_log_pos这个位置的日志继续同步。而这个位置就是所说的同步位点，也就是主库对应的文件名和日志偏移量

找同步位点很难精确取到，只能取一个大概位置。一种去同步位点的方法是这样的：

1.等待新主库A’把中转日志全部同步完成

2.在A’上执行show master status命令，得到当前A’上最新的File和Position

3.取原主库A故障的时刻T

4.用mysqlbinlog工具解析A’的File，得到T时刻的位点，这个值就可以作为$master_log_pos

这个值并不精确，有这么一种情况，假设在T这个时刻，主库A已经执行完成了一个insert语句插入了一行数据R，并且已经将binlog传给了A’和B，然后在传完的瞬间主库A的主机就掉电了。那么，这时候系统的状态是这样的：

1.在从库B上，由于同步了binlog，R这一行已经存在

2.在新主库A’上，R这一行也已经存在，日志是写在master_log_pos这个位置之后的

3.在从库B上执行change master命令，指向A’的File文件的master_log_pos位置，就会把插入R这一行数据的binlog又同步到从库B去执行，造成主键冲突，然后停止tongue

通常情况下，切换任务的时候，要先主动跳过这些错误，有两种常用的方法

一种是，主动跳过一个事务
	set global sql_slave_skip_counter=1;
	start slave;



另一种方式是，通过设置slave_skip_errors参数，直接设置跳过指定的错误。这个背景是，我们很清楚在主备切换过程中，直接跳过这些错误是无损的，所以才可以设置slave_skip_errors参数。等到主备间的同步关系建立完成，并稳定执行一段时间之后，还需要把这个参数设置为空，以免之后真的出现了主从数据不一致，也跳过了

## 2、GTID
MySQL5.6引入了GTID，是一个全局事务ID，是一个事务提交的时候生成的，是这个事务的唯一标识。它的格式是：

	GTID=source_id:transaction_id
- source_id是一个实例第一次启动时自动生成的，是一个全局唯一的值
- transaction_id是一个整数，初始值是1，每次提交事务的时候分配给这个事务，并加1

GTID模式的启动只需要在启动一个MySQL实例的时候，加上参数gtid_mode=on和enforce_gtid_consistency=on就可以了

在GTID模式下，每个事务都会跟一个GTID一一对应。这个GTID有两种生成方式，而使用哪种方式取决于session变量gtid_next的值

1.如果gtid_next=automatic，代表使用默认值。这时，MySQL就把GTID分配给这个事务。记录binlog的时候，先记录一行SET@@SESSION.GTID_NEXT=‘GTID’。把这个GTID加入本实例的GTID集合

2.如果gtid_next是一个指定的GTID的值，比如通过set gtid_next=‘current_gtid’，那么就有两种可能：

- 如果current_gtid已经存在于实例的GTID集合中，接下里执行的这个事务会直接被系统忽略
- 如果current_gtid没有存在于实例的GTID集合中，就将这个current_gtid分配给接下来要执行的事务，也就是说系统不需要给这个事务生成新的GTID，因此transaction_id也不需要加1

一个current_gtid只能给一个事务使用。这个事务提交后，如果要执行下一个事务，就要执行set命令，把gtid_next设置成另外一个gtid或者automatic

这样每个MySQL实例都维护了一个GTID集合，用来对应这个实例执行过的所有事务

## 3、基于GTID的主备切换
在GTID模式下，备库B要设置为新主库A’的从库的语法如下：
	CHANGE MASTER TO 
	MASTER_HOST=$host_name 
	MASTER_PORT=$port 
	MASTER_USER=$user_name 
	MASTER_PASSWORD=$password 
	master_auto_position=1 

其中master_auto_position=1就表示这个主备关系使用的是GTID协议

实例A’的GTID集合记为set_a，实例B的GTID集合记为set_b。我们在实例B上执行start slave命令，取binlog的逻辑是这样的：

1.实例B指定主库A’，基于主备协议建立连接

2.实例B把set_b发给主库A’

3.实例A’算出set_a与set_b的差集，也就是所有存在于set_a，但是不存在于set_b的GTID的集合，判断A’本地是否包含了这个差集需要的所有binlog事务

- 如果不包含，表示A’已经把实例B需要的binlog给删掉了，直接返回错误
- 如果确认全部包含，A’从自己的binlog文件里面，找出第一个不在set_b的事务，发给B

4.之后从这个事务开始，往后读文件，按顺序取binlog发给B去执行

## 4、GTID和在线DDL

如果是由于索引缺失引起的性能问题，可以在线加索引来解决。但是，考虑到要避免新增索引对主库性能造成的影响，可以先在备库加索引，然后再切换，在双M结构下，备库执行的DDL语句也会传给主库，为了避免传回后对主库造成影响，要通过set sql_log_bin=off关掉binlog，但是操作可能会导致数据和日志不一致

两个互为主备关系的库实例X和实例Y，且当前主库是X，并且都打开了GTID模式。这时的主备切换流程可以变成下面这样：

- 在实例X上执行stop slave
- 在实例Y上执行DDL语句。这里不需要关闭binlog
- 执行完成后，查出这个DDL语句对应的GTID，记为source_id_of_Y:transaction_id
- 到实例X上执行一下语句序列：

	set GTID_NEXT="source_id_of_Y:transaction_id";
	begin;
	commit;
	set gtid_next=automatic;
	start slave;

这样做的目的在于，既可以让实例Y的更新有binlog记录，同时也可以确保不会在实例X上执行这条更新

# 七、MySQL读写分离
读写分离的基本结构如下图：
![Nf4hvt.png](https://s1.ax1x.com/2020/06/29/Nf4hvt.png)
读写分离的主要目的就是分摊主库的压力。上图中的结构是客户端主动做负载均衡，这种模式下一般会把数据库的连接信息放在客户端的连接层。由客户端来选择后端数据库进行查询

还有一种架构就是在MySQL和客户端之间有一个中间代理层proxy，客户端只连接proxy，由proxy根据请求类型和上下文决定请求的分发路由

![Nf4jvq.png](https://s1.ax1x.com/2020/06/29/Nf4jvq.png)

1.客户端直连方案，因此少了一层proxy转发，所以查询性能稍微好一点，并且整体架构简单，排查问题更方便。但是这种方案，由于要了解后端部署细节，所以在出现主备切换、库迁移等操作的时候，客户端都会感知到，并且需要调整数据库连接信息。一般采用这样的架构，一定会伴随一个负责管理后端的组件，比如Zookeeper，尽量让业务端只专注于业务逻辑开发

2.带proxy的架构，对客户端比较友好。客户端不需要关注后端细节，连接维护、后端信息维护等工作，都是由proxy完成的。但这样的话，对后端维护团队的要求会更高，而且proxy也需要有高可用架构

在从库上会读到系统的一个过期状态的现象称为过期读

## 1、强制走主库方案
强制走主库方案其实就是将查询请求做分类。通常情况下，可以分为这么两类：

1.对于必须要拿到最新结果的请求，强制将其发到主库上

2.对于可以读到旧数据的请求，才将其发到从库上

这个方案最大的问题在于，有时候可能会遇到所有查询都不能是过期读的需求，比如一些金融类的业务。这样的话，就需要放弃读写分离，所有读写压力都在主库，等同于放弃了扩展性

## 2、Sleep方案
主库更新后，读从库之前先sleep一下。具体的方案就是，类似于执行一条select sleep(1)命令。这个方案的假设是，大多数情况下主备延迟在1秒之内，做一个sleep可以很大概率拿到最新的数据

以买家发布商品为例，商品发布后，用Ajax直接把客户端输入的内容作为最新商品显示在页面上，而不是真正地去数据库做查询。这样，卖家就可以通过这个显示，来确认产品已经发布成功了。等到卖家再刷新页面，去查看商品的时候，其实已经过了一段时间，也就达到了sleep的目的，进而也就解决了过期读的问题

但这个方案并不精确：

1.如果这个查询请求本来0.5秒就可以在从库上拿到正确结果，也会等1秒

2.如果延迟超过1秒，还是会出现过期读

## 3、判断主备无延迟方案
show slave status结果里的seconds_behind_master参数的值，可以用来衡量主备延迟时间的长短

1.第一种确保主备无延迟的方法是，每次从库执行查询请求前，先判断seconds_behind_master是否已经等于0。如果还不等于0，那就必须等到这个参数变为0才能执行查询请求

show slave status结果的部分截图如下：
![Nf5ZKx.png](https://s1.ax1x.com/2020/06/29/Nf5ZKx.png)

2.第二种方法，对比位点确保主备无延迟：

- Master_Log_File和Read_Master_Log_Pos表示的是读到的主库的最新位点
- Relay_Master_Log_File和Exec_Master_Log_Pos表示的是备库执行的最新位点

如果Master_Log_File和Read_Master_Log_Pos和Relay_Master_Log_File和Exec_Master_Log_Pos这两组值完全相同，就表示接收到的日志已经同步完成

3.第三种方法，对比GTID集合确保主备无延迟：

- Auto_Position=1表示这堆主备关系使用了GTID协议
- Retrieved_Gitid_Set是备库收到的所有日志的GTID集合
- Executed_Gitid_Set是备库所有已经执行完成的GTID集合

如果这两个集合相同，也表示备库接收到的日志都已经同步完成

4.一个事务的binlog在主备库之间的状态：

1）主库执行完成，写入binlog，并反馈给客户端

2）binlog被从主库发送给备库，备库收到

3）在备库执行binlog完成

上面判断主备无延迟的逻辑是备库收到的日志都执行完成了。但是，从binlog在主备之间状态的分析中，有一部分日志，处于客户端已经收到提交确认，而备库还没收到日志的状态
![Nf51Gd.png](https://s1.ax1x.com/2020/06/29/Nf51Gd.png)

这时，主库上执行完成了三个事务trx1、trx2和trx3，其中：

trx1和trx2已经传到从库，并且已经执行完成了
trx3在主库执行完成，并且已经回复给客户端，但是还没有传到从库中
如果这时候在从库B上执行查询请求，按照上面的逻辑，从库认为已经没有同步延迟，但还是查不到trx3的

## 4、配合semi-sync
要解决上面的问题，就要引入半同步复制。semi-sync做了这样的设计：

1.事务提交的时候，主库把binlog发送给从库

2.从库收到binlog以后，发回给主库一个ack，表示收到了

3.主库收到这个ack以后，才能给客户端返回事务完成的确认

如果启用了semi-sync，就表示所有给客户端发送过确认的事务，都确保了备库已经收到了这个日志

semi-sync+位点判断的方案，只对一主一备的场景是成立的。在一主多从场景中，主库只要等到一个从库的ack，就开始给客户端返回确认。这时，在从库上执行查询请求，就有两种情况：

1.如果查询是落在这个响应了ack的从库上，是能够确保读到最新数据

2.但如果查询落到其他从库上，它们可能还没有收到最新的日志，就会产生过期读的问题

判断同步位点的方案还有另外一个潜在的问题，即：如果在业务更新的高峰期，主库的位点或者GTID集合更新很快，那么上面的两个位点等值判断就会一直不成立，很有可能出现从库上迟迟无法响应查询请求的情况

![Nf5UZ8.png](https://s1.ax1x.com/2020/06/29/Nf5UZ8.png)

上图从状态1到状态4，一直处于延迟一个事务的状态。但是，其实客户端是在发完trx1更新后发起的select语句，我们只需要确保trx1已经执行完成就可以执行select语句了。也就是说，如果在状态3执行查询请求，得到的就是预期结果了

semi-sync配合主备无延迟的方案，存在两个问题：

1.一主多从的时候，在某些从库执行查询请求会存在过期读的现象

## 5、等主库位点方案
	select master_pos_wait(file, pos[, timeout]);

这条命令的逻辑如下：

1.它是在从库执行的

2.参数file和pos指的是主库上的文件名和位置

3.timeout可选，设置为正整数N表示这个函数最多等待N秒

这个命令正常返回的结果是一个正整数M，表示从命令开始执行，到应用完file和pos表示的binlog位置，执行了多少事务

1.如果执行期间，备库同步线程发生异常，则返回NULL

2.如果等待超过N秒，就返回-1

3.如果刚开始执行的时候，就发现已经执行过这个位置了，则返回0

![Nf5rzn.png](https://s1.ax1x.com/2020/06/29/Nf5rzn.png)

对于上图中先执行trx1，再执行一个查询请求的逻辑，要保证能够查到正确的数据，可以使用这个逻辑：

1.trx1事务更新完成后，马上执行show master status得到当前主库执行到的File和Position

2.选定一个从库执行查询语句

3.在从库上执行select master_pos_wait(file, pos, 1)

4.如果返回值是>=0的正整数，则在这个从库执行查询语句

5.否则，到主库执行查询语句

流程如下：
![Nf56s0.png](https://s1.ax1x.com/2020/06/29/Nf56s0.png)

## 6、GTID方案
	select wait_for_executed_gtid_set(gtid_set, 1);
这条命令的逻辑如下：

1.等待，直到这个库执行的事务中包含传入的gtid_set，返回0

2.超时返回1

等主库位点方案中，执行完事务后，还要主动去主库执行show master status。而MySQL5.7.6版本开始，允许在执行完更新类事务后，把这个事务的GTID返回给客户端，这样等GTID的方案可以减少一次查询

等GTID的流程如下：

1.trx1事务更新完成后，从返回包直接获取这个事务的GTID，记为gtid1

2.选定一个从库执行查询语句

3.在从库上执行 select wait_for_executed_gtid_set(gtid1, 1);

4.如果返回值是0，则在这个从库执行查询语句

5.否则，到主库执行查询语句
![Nf5WoF.png](https://s1.ax1x.com/2020/06/29/Nf5WoF.png)
