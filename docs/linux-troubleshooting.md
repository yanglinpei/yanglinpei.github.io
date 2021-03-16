
### 内存泄漏（表现：可用内存持续减少/OOM/进程内存消耗不多但系统内存不够/...）
    查看整体情况：syslog，TOP/ps观察进程RES，/proc/meminfo观察整体内存情况
        RES 高而 SHR 不高：可能堆内存泄漏，检查代码
        SHR 很高：tmpfs/shm持续增长，排查tmpfs/shm的使用
        VIRT 很高而 RES 很小，可能进程申请的虚拟地址空间没用到，虚拟地址空间泄漏，检查代码
        ...
    能看到问题进程
        pmap或/proc/pid/smaps观察具体哪块内存占用高。heap太大可能堆泄露；进程空间vma多，可能mmap多没有munmap；观察哪块持续增长
        pidstat跟踪，观察时间上是否有规律
        strace，grep mmap/brk/内存大小 ，看申请了用在哪里
        valgrind / memleak
    不能看到问题进程：观察/proc/meminfo异常项
        Active(anon)和Inactive(anon)大（只可以交换到swap的不能回收的内存，包括Shmem），从内存占用最大的进程开始，查看 pmap，排查程序malloc()申请的匿名内存
        AnonPages，没有对应文件的内存，不包括Shmem。排查代码malloc()匿名内存
        Mapped：mmap(2)系统调用方式申请还没有unmap的，排查代码mmap()
        Unevictable大，锁住了，不能被回收，查这些调用：ramdisk/ramfs；shm_lock申请的shmem；mlock()
        Mlocked，也属于Unevictable，排查代码mlock()
        Shmem占用多，看进程SHR排序，pmap，如果没有进程SHR大，看tmpfs里的文件。解决：限制tmpfs使用
        slab多，slabtop，排查驱动程序以kmalloc()申请的内存
        hugepage高：使用大页多，不会在free等命令中统计，不能交换，容易造成碎片，检查代码大页使用
        内核内存泄漏：/proc/meminfo 观察KernalStack、VmallocUsed和SUnreclaim大或持续增长不下降
            tracepoint：kmalloc, kfree, kmem_cache_alloc, kmem_cache_free等都在/sys/kernel/debug/tracing/events/，如果追踪不到，能判断出系统函数，用kprobe/systemtap追踪
            kmemleak：echo scan > /sys/kernel/debug/kmemleak有性能损耗，线下环境用
    pagecache回收失败导致内存不足：参考IO部分的pagecache部分
 
 
 ### IO问题（表现：系统卡顿/应用性能抖动/IO高/load高/...）
 
    整体情况：syslog，TOP iowait, iotop, iostat 看avserv/avwait/avque, pidstat -d找到问题进程，跟踪strace -fp pid, lsof , opensnoop，一般原因：
        大量读写文件
        应用程序使用相关，如mysql慢sql，redis频繁快照等
        page cache相关问题
    page cache问题（内核管理， 过多控制有得有失，慢慢调整，看应用反应）：
        判断是否由page cache引起：sar -B的pgscan变化大或/proc/pressure/memory信息avg10>40（4.20以上内核）。
        具体是哪种问题：
            可采集/proc/vmstat 观察指标变化来做大致判断：如compact_fail表示系统内存碎片严重，需调整碎片指数或手动内存规整
            太多直接内存回收：sar -B 有pgscand，调整水位vm.min_free_kbytes早点使用内核回收（参考值128G内存调整为4G）
            脏页过多：调整radio让脏页在适当的比例
            NUMA配置：vm.zone_reclaim_mode = 0 允许使用其他node的内存
            过度回收：grep inodesteal /proc/vmstat 是否存在过度回收：对page cahce敏感的业务，重要数据mlock(2)防止被回收，可回收数据主动madvise(2)立即回收/ 用cgroup控制
        具体由什么引起的？
            静态追踪（预置了追踪点），如追踪直接回收的tracepoint： mm_vmscan_direct_reclaim_begin(/end) ，碎片整理mm_compaction_begin(/end)
            动态追踪（需要借助 probe）perf/ systemtap/ ebpf


### 网络问题（表现：抖动，延迟，丢包）
    整体情况：syslog/netstat -s/ss -s/dstat 查看报告，判断哪层可能有问题，再分层排查：tcpdump/tracepoint
    TCP层
        建立连接时：
            client调用connect()发送SYN，可能没收到回复。解决：重试次数设置：tcp_syn_retries =2 
            server收到SYN，半连接队列满：SYNs to LISTEN sockets dropped/kernel: [3649830.269068] TCP: Possible SYN flooding on port xxx. Sending cookies. Check SNMP counters。解决：tcp_max_syn_backlog半连接队列长度设大。防止SYN FLOOD记录连接cookie：net.ipv4.tcp_syncookies = 1
            server发送的SYNACK没收到回复。解决：重传net.ipv4.tcp_synack_retries = 2
            三次握手完毕连接建立，全连接队列满丢包：-xxx times the listen queue of a socket overflowed。解决：net.core.somaxconn = 16384。处理不掉会丢弃但是不发RST， net.ipv4.tcp_abort_on_overflow = 0
        断开连接时：
            FIN_WAIT_1状态过多：netstat统计。解决：数量限制tcp_max_orphans，重试限制tcp_orphan_retries
            FIN_WAIT_2状态过多：netstat统计。解决：超时时间tcp_fin_timeout=2，重试限制tcp_orphan_retries
            TIMEWAIT过多：time wait bucket table overflow。解决：减小TIME_WAIT最大个数 net.ipv4.tcp_max_tw_buckets = 10000；复用net.ipv4.tcp_tw_reuse = 1，作为发起方有用
            CLOSE_WAIT多：netstat统计。解决：检查代码
            LAST_ACK过多：netstat统计。解决：重试限制tcp_orphan_retries
        TCP收发包
            重传：netstat -s|grep retransmit。解决：tracepoint观察echo 1 > tcp/tcp_retransmit_skb/enable
            TLP+延迟ack导致无效重传：netstat -s |grep TCPLossProbes。解决：修改 tcp_early_retrans，客户端打开TCP_QUICKACK/服务端使用TCP_NODELAY禁用nagle算法
            内存不足：out of memory -- consider tuning tcp_mem/8094 packet reassembles failed
                收包溢出：netstat -s|grep "packet receive errors"。解决：设置合理socket缓冲区参数net.core.rmem_default ， net.core.rmem_max
                发包溢出：netstat -s|grep "send buffer errors"。解决：设置合理socket缓冲区参数net.core.wmem_default， net.core.wmem_max
                TCP发送缓冲区，net.ipv4.tcp_wmem = min default max， max不能超过net.core.wmem_max = 16777216 （tracepoint sk_stream_wait_memory）
                TCP接收缓冲区，net.ipv4.tcp_rmem = min default max 可关闭自动调节net.ipv4.tcp_moderate_rcvbuf/setsockopt() 中的配置选项 SO_RCVBUF
                总内存限制 net.ipv4.tcp_mem = 8388608 12582912 16777216 单位页（tracepoint sock_exceed_buf_limit）
            乱序丢包：tcpdump抓包是否有很多乱序丢包。解决：net.ipv4.tcp_recordering=3
            带宽利用率低：解决：长连接考虑关闭tcp_slow_start_after_idle。根据业务场景和网络条件使用适合的算法，丢包多不想影响网络吞吐拥塞控制用BBR， 延迟大调缓存
    UDP层
        netstat -s grep Udp
        调整缓冲区net.ipv4.udp_mem， net.ipv4.udp_rmem_min， net.ipv4.udp_wmem_min
    网络层
        本地接口对应IP配置有误：查看路由表ip r show table local。解决：正确配置IP/补上路由条目
        路由配置有误：netstat -s|grep "missing route"/ip r get 目标IP。解决：配置正确路由
        反向路由rp_filter验证：cat /proc/sys/net/ipv4/conf/eth0/rp_filter。解决：配置成0（不验证）或2（松散模式）
        防火墙
            规则配置有误，修改防火墙规则
            ip_conntrack跟踪表：dmesg报错ip_conntrack: table full, dropping packet。解决：增大net.netfilter.nf_conntrack_max，减少跟踪的有效时间net.netfilter.nf_conntrack_tcp_timeout_established = 1200 & net.netfilter.nf_conntrack_udp_timeout_stream = 180 & net.netfilter.nf_conntrack_icmp_timeout = 30
        分片相关
            超时：601 fragments dropped after timeout。解决：net.ipv4.ipfrag_time = 30，sysctl -w net.ipv4.ipfrag_time=60
            超过分片内存帽，安全检查丢包/分片安全距离检查丢包：8094 packet reassembles failed。解决：调整net.ipv4.ipfrag_high_thresh & net.ipv4.ipfrag_low_thresh / 关闭ipfrag_max_dist
            分片哈希表冲突链超过默认128：inet_frag_find: Fragment hash bucket 128 list length grew over limit. Dropping fragment. 解决：内核补丁
        端口不够：Cannot assign requested address。解决：端口范围调整 net.ipv4.ip_local_port_range = 1024 65535。
    链路层
        arp_ignore配置导致不回arp包
        arp_filter配置打开了导致可能不回arp包
        arp表满，调大 net.ipv4.neigh.default.gc_thresh3
        arp缓存队列溢出，cat /proc/net/stat/arp_cache ，unresolved_discards是否新增。调整unres_qlen_bytes
    网卡层
        Ring buffer填满。ethtool -S eth0|grep rx_fifo。解决：修改缓冲区大小ethtool -G eth0 rx 4096 tx 4096
        qdisc队列满：ifconfig/ip -s -s link ls dev eth1  dropped不为0。解决：调整disc的队列长度txqueuelen：ifconfig eth0 txqueuelen 2000
        端口协商速率不统一：查看网卡丢包ethtool -S eth0；查看两端网卡配置ethtool eth0。解决：重新自协商 ethtool -r  eth0；强制设置速率
        网卡流控pause桢：ethtool -S eth1 | grep control不为0。解决：关闭流控
        mac地址错误丢包，因arp配置等，导致mac地址学习失败：抓包看mac地址是否正确。解决：触发arp重新学习
        大包丢包：ethtool -S eth0|grep lenth_error。解决：MTU配置支持大包
        包从网卡到内核栈之间的缓冲队列溢出丢包：/proc/net/softnet_stat。解决：设置net.core.netdev_max_backlog
        多核CPU使用不均，中断处理不过来：TOP， mpstat单核使用率高。解决：调整RSS队列配置/关闭RPS/调整中断平衡/调整NUMA配置
        收包一次轮询多个包：解决：net.core.netdev_budget = 600


### CPU（表现：系统卡顿/应用性能抖动/...）
    syslog, TOP看哪种CPU使用高：sys /us / wa/ hi /si。strace/perf->ftrace->tracepoint/kprobe
        sys高， perf/ftrace排查时间段的，瞬时的可以用sysrq看。上述内存/网络/IO中的问题很多都有可能是CPU被内核使用率高的原因，比如
            内存碎片多cat /proc/vmstat|grep compact_fail，触发了直接内存规整。如使用大页THP，grep -i HugePages /proc/meminfo。关闭echo never > /sys/kernel/mm/transparent_hugepage/enabled / 
            直接内存清理多，参考page cache部分
            mysql高并发，慢sql：sql优化索引优化/微调innodb_spin_wait_delay
            大量IO，参考IO部分
            并发网络连接太多/timewait高/端口耗尽等，参考网络部分
            ...
        us高，strace/perf/USDT， 内核问题ftrace
        中断 /proc/interrupts 和/proc/softirqs。 tracepoint。常见并发网络连接太多
        wa高，iostat查看IO的实际量，高才能认为是IO问题，参考IO部分
        上下文切换：vmstat/pidstat -w（-t查看线程） 查看上下文切换，cswch自愿切换高，I/O、内存等系统资源不够；nvcswch非自愿切换高，确实CPU不够用了
        找不到高CPU使用的进程：短时进程pstree跟踪父进程/ftrace跟踪进程调度sched_switch
