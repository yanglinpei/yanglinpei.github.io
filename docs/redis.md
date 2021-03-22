REDIS特性	三高一丰富																								
高性能																									
丰富数据类型																									
内存数据库																									
缓存																									
REDIS高可用高可扩展	主从，切片																								
																									
键值数据库对比	内存/磁盘	应用接口（数据结构）	持久化	部署架构	访问方式	可靠性	性能																		
REDIS	内存	5类	持久化	集群	c/s	可靠	ms, 100ns																		
Memcached	内存	string	内存	集群	c/s	内存	ms, 100ns																		
MongoDB	磁盘	json结构化	持久化	集群	c/s	可靠	ms, 基于磁盘																		
RocksDB	磁盘	kv	持久化	单实例	lib库	lib	ms, 基于磁盘																		
																									
																									
																									
功能模块视图																									
数据结构																									
内存管理																									
持久化																									
系统监控																									
切片	事件驱动																								
主从																									
哨兵																									
																									
源码怎么看																									
数据结构																									
adlist.c	双向链表																								
geoxxx	地理位置																								
intset.c	整数集																								
sds	简单动态字符串																								
ziplist																									
zipmap																									
dict																									
rax	redistree																								
aexxx	网络通信																								
anet	TCP																								
aof.c	AOF持久化																								
rdb.c	RDB持久化																								
evict.c	缓存淘汰策略，LFULogIncr																								
expire.c	缓存																								
replication.c	主从																								
sentinel	哨兵																								
cluster.c	集群																								
latency	系统监控																								
																									
技术栈																									
数据存取	基本数据结构	不同数据结构选择																							
		空间与时间均衡优化																							
	内存管理																								
数据持久化	RDB快照																								
	AOF日志																								
可用性	主从																								
	哨兵																								
扩容	数据分布																								
	数据迁移	客户端感知的迁移																							
	集群规模	集群通信机制																							
缓存应用	缓存原理																								
	缓存策略																								
	缓存一致性																								
	缓存异常																								
性能优化	线程模型																								
	阻塞操作																								
																									
数据类型																									
k：字符串																									
v：多种类型	序列化：多块内存地址放到一起																								
如果只支持string类型，	存储需要做序列化，还要加分隔符																								
	读取需要反序列化																								
全局哈希表	整个就是个大哈希表，表项：dictEntry																								
保存所有kv对	hash(k)%size																								
实现ht1和ht2，渐进式rehash																									
																									
哈希冲突	chain hash																								
解决rehash（阻塞）	新的hash函数/二次hash																								
																									
渐进式rehash：																									
触发条件	负载因子de/size																								
约束条件																									
操作时机	伴随正常操作																								
	周期性																								
																									
数据类型	底层	场景																							
String	二进制安全的字节数组	数值，文本，图片																							
List	双向链表，支持双向pop，push		场景：排行榜，关注列表																						
Hash	hash字典	结构化数据																							
Set	无须集合，去重	共同关注																							
Sorted Set	有序集合，去重，支持集合操作	有序排行榜，有序统计																							
bitmap	二进制安全的字节数组，位操作，底层SDS																								
hyperloglog	基数统计	日活月活																							
																									
																									
																									
																									
底层数据结构redisObject	server.h定义																								
key，value都是redisObject																									
type：string,list,hash,set,zset，根据类型调用不同API	32bit = 4B																								
encoding编码方式，节省内存，例如
hash底层可以是HT,ZIPLIST,QUICKLIST
string可以是emdstr（44B内），raw									这个单位是bit																
lru																									
refcount引用计数	4B																								
ptr指针指向实际key或value的指针，string就是指向SDS	8B	如果是整形，直接存数，不是存指针																							
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
dictentry 24B向上取整变32B（根据makefile里的malloc种类不一样）	一共64B																								
key 4B+4B+8B = 16B																									
value 4B+4B+8B = 16B																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
实战问题	hyperloglog, bitmap, set																								
数据结构实战一：一亿个ID对用什么类型保存	100010001 200020002																								
分析：元数据类型为基本的kv，v是单值，适合string																									
方案一，string，浪费空间，元数据空间，对比hash方式																									
																									
方案二：hash可使用压缩列表ziplist，quicklist	拼接存储
10001做k，0001做v
v里面是kv对，存在entry里																								
ZIPLIST（压缩列表）共享元数据结构和redisObject元数据，省空间，但是查询不好查																									
有个参数限制最大元素数量和单个元素最大值																									
																									
list，hash，sorted Set都有紧凑型内存结构哦																									
																									
数据结构实战二：电商网站UV统计																									
分析：同一天只记为1次（去重）																									
方案一：set																									
pageID:uv为k，userID为value																									
sadd不会重复记录																									
方案二：hash																									
pageID:uv为k，userID为集合的key，1为hash集合的value																									
HSET添加记录，不会重复																									
方案三：hyperloglog计数统计																									
pageID:uv为k，userID为value																									
使用PFADD不会重复																									
优势节省内存																									
不足：有误差																									
																									
数据结构实战三：移动应用签到功能，如何统计千万用户一个月内连续签到N天的情况																									
分析：每个用户一天签到了就用1位表示																									
使用string或集合开销过大																									
方案：bitmap 按位操作																									
用户ID加月份作为key，整月的签到情况作为value																									
SETBIT																									
GETBIT																									
BITCOUNT																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
数据结构实战四：统计社交网站的用户好友	使用set，取交集	使用set，取交集																							
方案：使用set，用户id作为set集合的key，用户好友userid作为value，使用交集找到共同好友																									
如何统计新增用户和留存用户	差集是新增，交集是留存																								
一天的用户为一个set																									
几天的差集是新增，交集是留存	client.sinter(set1, set2)	交集
client.sunion(set1, set2)	并集
client.sdiff(set1, set2)	差集																								
																									
怎么选数据类型：功能性选择																									
string	通用，二进制安全																								
hash	结构化数据优先选择，还支持value的拆分保存节省内存																								
list	FIFO序存取数据																								
set	去重，集合																								
sorted set	带权重排序需求的																								
																									
怎么选数据类型：性能选择																									
string	直接查询全局哈希表O(1)																								
集合类型	查完全局哈希表后再查找集合，多一次查找
集合操作一般为O(N)																								
																									
																									
怎么选数据类型：空间选择																									
string	对于小数据，SDS的额外开销大																								
集合类型ziplist，1000个元素10kb	list, hash 和sorted set可以使用紧凑型内存结构节省空间																								
																									
																									
高可用架构	可靠性 VS 性能																								
																									
数据可用性：																									
1. 实例异常退出时，数据不丢失																									
2. 持久化保存数据：																									
内存快照RDB - Redis DB	创建	fork机制																							
	恢复																								
日志AOF - Append-Only File	落盘																								
	恢复																								
	重写																								
																									
实例可用性：	问题：
主从一致吗
故障切换失败	CAP																							
1. 一个实例异常退出，还有其他实例能服务																									
2. 主从集群																									
主从																									
哨兵+故障切换																									
																									
数据可用性：																									
RDB																									
存的是那个时刻的内存镜像（所以实例内存不要过大4-6G，配置项maxmemory），fork陷入内存，阻塞了用户态																									
fork出的子进程将内存镜像写在rdb文件																									
bgsave																									
恢复：将rdb文件拷贝到redis数据目录，启动redis-server																									
使用RDB恢复会过滤掉过期数据																									
AOF																									
存实例接收到的所有写操作																									
aof文件，追加写磁盘																									
三种配置落盘时机，no（交给内核，30s）, everysec, always																									
恢复：将aof文件拷贝到redis数据目录，启动redis-server																									
重写：同样的key的重复操作比如set多次，去掉，会影响性能	会需要分配内存空间，如果同时有很多写入。容易OOM																								
																									
4.0以前，同时配置了RDB和AOF	混合配置的目的是节省空间，aof比较不占空间																								
优先AOF					 																				
																									
4.0混合使用：两个时刻做RDB中间做AOF																									
T1 RDB																									
AOF																									
T2 RDB																									
																									
高可用实战一：一台服务器三个Redis实例，内存共32G，每个实例10G，使用RDB,实例响应有卡顿，还OOM																									
分析：																									
RDB使用fork创建内存快照																									
fork的执行时间和实例内存大小有关																									
如果有大量写 COW，需要新分配内存																									
																									
																									
实例可用性																									
主从	全量复制	RDB，复制缓冲区																							
	增量复制	复制积压缓冲区																							
哨兵：自动化故障切换	心跳与下线判断	主客观下线	不是redis的，了解：可以网络心跳+磁盘心跳																						
		quorum机制																							
	故障切换	哨兵leader选举	分布式共识RAFT																						
		选主库																							
		主从切换																							
																									
主从间第一次复制																									
1. 主库通过全量复制把RDB发给从																									
2. 全量复制完成后，将复制阶段新的写操作发给从																									
3. 主库可以读写，从库只读																									
																									
主从间增量复制（网络断连）																									
1. 正常情况主库收到新的写 就发送给从库																									
2. 网络断连，从库再次连接后，执行增量复制																									
3. 从库复制偏移量在复制挤压缓冲区中，就可以发送																									
4. 如果没有，要全量复制																									
																									
																									
																									
																									
																									
哨兵机制																									
主库故障怎么判断：哨兵监控主和从的心跳，需要客观下线																									
主观下线：哨兵自行判断																									
客观下线：超过quorum个哨兵判断主观下线	quorum一般设为n/2+1，n是哨兵个数，也可以设为n																								
																									
主库故障谁来处理，先选举一个哨兵leader																									
1. 通过哨兵进行主从切换																									
2. 哨兵leader选举																									
																									
谁发现客观下线，谁发起命令，并给自己投一票，否决其他票																									
其他哨兵投收到的第一个请求																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
高可用实战二：主从异常：从库读不到最新数据																									
分析：redis不提供强一致性																									
解决：																									
1. 只读主库																									
2. 判断从库offset，落后太多就读主库																									
																									
高可用实战三：在从库读到过期数据																									
分析：																									
过期命令使用相对时间expire T1+XXX，在主从复制的过程中，从库有可能读到过期数据																									
3.2 以前不检查过期， 3.2以后对过期数据返回空																									
使用expireat + 绝对时间																									
																									
高可用实战四：主库里的数据丢了，客户端写入成功但读不到																									
分析：曾发生主从切换，仍有客户端把数据写到旧主库																									
原因：																									
旧主库临时hang住，被判断为客观下线，哨兵开始主从切换																									
切换未完成时，旧主库恢复，客户端在写入																									
新主库上线，旧主库执行主从复制会清空自己数据																									
解决：让主库只读	关键点fence机制：判断旧主库有问题，要隔离他，免得他诈尸																								
min-slave-to-write: 主库至少跟几个从库同步（控制让主库不能写）																									
min-slave-max-log：主从复制时，从库给主库发送ack的最大延迟																									
																									
																									
REDIS性能优化：低延迟响应																									
ops - operation per second																									
latency	redis单线程，请求一个一个处理：
1/ops=latency																								
																									
redis快的原因：	相反的也就是导致慢的原因																								
内存操作	内存分配，内存复制																								
精巧的数据结构设计，比如zset，zscore。跳表+hash表（score，element）
扩展：范围查询（scan）+单点查询（set，get），都可以用两种索引来做，比如b+tree + hash表																									
异步化操作：耗时长的用多线程，fork子进程也是	bio.c																								
高效网络IO：epoll																									
																									
单线程架构																									
进程是一个程序启动起来，在OS看来是一个结构体记录了进程相关信息，比如页表，fd，权限等																									
真正在处理任务的时候，在内核看来是一个task，是可调度的																									
所以，进程通常是从资源的角度来看，然后真正处理任务的时候，是被内核看成一个task																									
																									
redis是一个单线程进程																									
请求处理，键值对操作，日志，由主线程完成。	异步线程完成耗时长的操作，比如网络，主从复制																								
																									
单线程优势：开发模型简单，避免切换，同步开销																									
																									
单线程不足：容易阻塞																									
																									
事件驱动框架	文件事件，时间事件																								
单线程redis怎么弄																									
启动event数组，放的网络描述符，轮询	配置项maxclients
进程ulimit																								
																									
性能实战一：常见的阻塞操作有哪些																									
																									
网络IO	epoll	bigkey(10kb)打满网卡	升级网卡，bound网卡																						
请求操作	单线程	O(N)操作
https://redis.io/commands																							
磁盘IO	RDB使用fork；
AOF采用三种配置	大内存实例导致RDB慢
AOF everysec/always																							
主从同步	全量复制+增量复制	缓冲区溢出，复制失败																							
																									
异步操作																									
主线程fork子进程RDB操作	bgsave子进程																								
	bgrewriteaof子进程																								
	主从无盘复制子进程																								
主线程pthread_create子线程	AOF日志写																								
	惰性删除																								
	文件关闭																								
																									
BIGKEY																									
bigkey的排查																									
1. value很大的string类型	例如：很大的json字符串，比如缓存数据库一整条数据																								
2. 元素很多的集合类型	例如，社交网站粉丝列表，网站统计集合等																								
																									
排查方法																									
1. redis-cli --bigkeys --i 0.1																									
2. SCAN + memory usage																									
3. rdbtool：rdb -c memory -f res.csv (--bytes xxx) xxx.rdb	推荐																								
4. redis-memory-for-key key	单个key																								
																									
bigkey的处理																									
删除方法：																									
1. unlink																									
																									
缓冲区机制	风险	处理																							
输入缓冲区1g写死	接收bigkey；
服务端处理慢	避免bigkey																							
输出缓冲区，包括复制缓冲区	1. 返回bigkey；
	避免bigkey
																							
	2. 主从复制，网络不稳或从节点处理慢；	参数调整：
应⽤用客户端：
• client-output-buffer-limit normal 0 0 0
• Pubsub客户端：client-output-buffer-limit pubsub 8mb 2mb 60
• 主从复制时主节点客户端：client-output-buffer-limit slave 512mb 128mb 60																							
	3. monitor命令，结果占用输出缓冲区																								
复制缓冲区和复制积压缓冲区	都可以配置																								
																									
性能实战二：海量数据加载遇到长尾延迟（tail latency）																									
场景：数据从⼀一个Redis实例例迁移到另⼀一个实例例，加载亿级数据量量。																									
现象：Redis间歇性出现秒级延迟的请求，严重影响前端应⽤用访问数据。																									
分析：																									
• 有没有bigkey？																									
• 有没有启⽤用AOF、RDB？																									
排查：																									
• 持续写⼊入数据à 哈希表逐步增⼤大																									
à 发⽣生rehash																									
																									
性能实战三：上云后性能抖动																									
分析：																									
• 没有bigkey；																									
• 没有启⽤用AOF、RDB；																									
• 没有海海量量数据加载；																									
• 还有什什么原因？																									
排查：																									
• 同⼀一台服务器器上运⾏行行有其他CPU密集型程序；																									
• 导致Redis实例例受影响。																									
																									
性能实战四：OOM，有redis进程被kill																									
bigkey																									
AOF rewrite，重写时使用额外内存缓存新写入数据，刚好有很多新写入数据过大																									
解决：																									
设置AOF重写阈值：auto-aof-rewrite-percentage：当前的aof⽂文件⼤大⼩小超过上⼀一次rewrite后aof⽂文件的百分⽐比后触发rewrite																									
避免bigkey																									
																									
故障排查																									
CPU阻塞																									
• ⾼高复杂度键值对操作，例例如集合操作																									
• bigkey																									
• 程序混布时的⼲干扰																									
• 实例例过⼤大导致fork耗时过⻓长																									
内存溢出																									
• bigkey导致输⼊入缓冲区溢出																									
• 输出缓冲区溢出																									
• bigkey																									
• 主从复制过慢，新写⼊入量量过⼤大																									
• Monitor命令使⽤用不不当																									
• AOF重写时新写⼊入量量过⼤大导致内存溢出																									
磁盘IO阻塞																									
• AOF使⽤用always，everysec配置																									
• RDB实例例过⼤大，写盘缓慢																									
网络IO阻塞																									
• bigkey打满⽹网卡																									
																									
																									
五 REDIS 缓存	策略介绍，雪崩击穿穿透，一致性																								
缓存写满了时，替换方式：	WB写回：先写缓存，数据替换时写后端数据库 --- 可能丢数据，但更快
WT直写：写缓存同时写数据库																								
缓存替换策略：	按候选数据集区分：
volatile-xxx 比如按过期时间volatile-TTL，就不会包含没有设置过期时间的数据
allkeys-xxx 所有的，不区分过不过其																								
	按策略区分：
LRU：最近最少使用。scan会污染
LFU：最少频率使用。counter不是简单+1；短期大量访问后面不访问，lfu_decay_timer会做衰减
TTL：淘汰将要过期的
Random																								
																									
缓存类型	1. 只读（更多），会删除，只是不更新
2. 读写		腾大入职：v_lililuo(罗琍琍)、v_huitzhan(詹慧婷)																						
缓存污染	scan扫到了，但是没有做贡献																								
																									
思考题， memcached和redis																									
	数据类型	持久化	集群策略																						
memcached	string	不支持	一致性哈希																						
redis	丰富	支持	哈希槽																						
一致性哈希：节点变动频繁的时候，会减少变动，性能差不多																									
																									
思考题， 作为只读缓存，要考虑写回策略吗																									
不需要，作为只读时，最新数据都是数据库中；reids自行替换																									
																									
缓存实战一：缓存大量同时失效	雪崩-面																								
分析：大量cache miss；数据库压力剧增；缺失的数据再次更新到redis压力剧增；雪崩																									
应对：																									
1. 给数据设置不同的TTL值																									
2. 通过主从集群增加可靠性																									
																									
缓存实战二：某一个热点数据失效	击穿-点																								
现象：数据库qps增加																									
分析：不同于雪崩，仅少量数据失效，缓存击穿																									
应对方法：热点数据设置不同的过期时间																									
																									
缓存实战三：数据库误删数据	穿透-数据库也没了																								
场景：volatile-ttl，误删数据																									
现象：redis qps和mysql qps都突然增加																									
分析：redis数据过期，在数据库也找不到，缓存击穿																									
应对：																									
缓存缺失在db里查不到的数据，在redis设置null																									
使用 Bloom filter 让redis快速返回	bit数组，判断是否存在，3个hash函数，算出bit数组里的3个位置，如果有0则不存在，都是1不一定表示存在																								
redis和mysql超过告警，触发前端限流																									
																									
附加思考题：前端限流																									
Q: 前端限流对 雪崩，击穿，穿透都有用吗，如果是恶意访问，前端限流是好方案吗																									
A: 前端限流都建议部署； 是个有损方案，对于恶意访问，在更前端，网关的位置进行校验																									
																									
缓存实战四：缓存一致性如何保证	业务需要对数据更新																								
更新还是删除？	删除，更新要考虑一堆东西，要原子性，写多读少没必要更新缓存																								
两种方案：																									
1. 先删除缓存，再更新DB																									
2. 先更新DB，再更新缓存																									
																									
场景：当有一个操作执行失败了	并发过来时，哪个方案影响更小																								
方案一																									
• 先删除缓存，尚未更更新DB -> 有并发请求访问缓存 -> 缓存缺失 -> 从DB读取旧值																									
• 先删除缓存，更新DB失败 -> 重试DB更新 -> 有并发请求访问缓存 - > 缓存缺失 - > 从DB读取旧值																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
网络事件处理epoll老师讲的轮询是不是select的，能否详细讲讲																									
问题，rehash造成延迟的问题，怎么优化																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
																									
