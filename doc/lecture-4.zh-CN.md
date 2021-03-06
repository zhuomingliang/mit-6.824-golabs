6.824 2016讲座4：主/备份复制

今天
  1. 主/备份复制
    1. 广泛使用：例如，GFS块服务器
    2. 但今天是强一致性（“理想”）
    3. 这么重要，探索更深入
  2. VM-FT案例研究

容错
  1. 我们想要一个服务，尽管失败但仍能继续使用！
  2. 可用性：仍然可用尽管[某些]失败
  3. 强一致性：就像一个单一的服务器对客户端，比GFS提供的文件更强
  4. 很难！
  5. 很有用！

需要一个失败模型：我们将努力应对什么？
  1. 独立的故障-停止的计算机故障，VM-FT进一步假定一次只有一个故障
  2. 站点电源故障（并最终重新启动）
  3. 网络分区
  4. 没有bug，没有恶意

核心思想：复制
  1. *两个*服务器（或更多）
  2. 每个副本保持服务所需的状态
  3. 如果一个副本失败，其他副本可以继续

示例：容错MapReduce主机
  1. 实验一 worker 已经是容错的，但不是 master，它是一个“单点故障”
  2. 我们可以有两个 master，万一一个失败
  [图：M1，M2， worker ]
  1. 状态：
     1. worker 列表
     2. 哪些工作完成
     3. worker 空闲
     4. TCP连接状态
     5. 程序计数器

大问题：
  1. 什么状态复制？
  2. 副本如何获取状态？
  3. 什么时候切换备份？
  4. 在切割时是否出现异常？
  5. 如何修复/重新集成？

两种primary方法：
  1. 状态转移，“主”副本执行服务，primary向备份发送[新]状态
  2. 复制状态机
    1. 所有副本都执行所有操作
    2. 如果相同的开始状态，同样操作，相同的顺序，确定性，然后相同的结束状态

状态转移更简单
  1. 但状态可能很大，转移缓慢
  2. VM-FT使用复制状态机

复制状态机可以更高效
  1. 如果操作与数据相比较小
  2. 但复杂得到正确。例如，对多核，决定论。实验中使用复制状态机

什么是复制状态机？
  1. K/V的 put 和 get？
  2. x86指令？
  3. 影响性能和易于实现
    1. 在primary和备份之间需要多少交互，高级抽象机器可以传递较少的信息
    2. 处理非确定性在x86级别很难，但是在put / get级别更容易
  4. 更高级别的RSM仅适用于使用更高级别接口的应用程序
    - x86 RSM可以执行任何x86程序

容错虚拟机的实用系统的设计，Scales，Nelson和Venkitachalam，SIGOPS OSR Vol 44，No 4，2010年12月

非常雄心勃勃的系统：
  1. 全系统复制
  2. 对应用程序和客户端完全透明
  3. 任何现有软件的高可用性
  4. 如果它工作得很好将是多么神奇！
  5. 故障模型：
    1. 独立硬件故障
    2. 站点电源故障
  1. 仅限于单处理器虚拟机
    问：多核处理器的难点是什么？

概述
  [diagram：app，O / S，VM-FT underneath]
  1. 两台机器，主备; 和其他机器
  2. 两个网络：客户端到服务器，日志记录通道
  3. 共享磁盘用于永久存储
  4. 主备之间用“lock step”
    1. primary将所有输入发送到备份
    2. 丢弃备份的输出
  5. 心跳在主备之间
    1. 如果主失败，启动备份执行！

什么是输入？
  1. 时钟中断
  2. 网络中断
  3. 磁盘中断

挑战：
  1.使它看起来像一个可靠的服务器
    1. 如果主失败和副本接管了，外部世界会看到什么？
      1. 如果 primary 在向客户端发送响应之前或之后失败，可能会丢失客户端请求？执行两次？
    2. 主服务器何时向客户端发送响应？
  2.如何避免两个 primary ？（“裂脑综合征”）
    1. 如果日志记录通道断开怎么办？
    2. primary和备份是否都是 primary？
  3.如何使备份成为 primary 的副本
    1. 什么操作必须发送到备份？
      - 时钟中断？
    1. 如何处理非确定性？
      - 例如，中断必须在备份时以与primary相同的指令传送

挑战3解决方案：确定性重放
  1. 目标：使x86平台确定性
    1. 想法：使用虚拟机管理程序使虚拟x86平台确定性
    2. 两个阶段：记录和重放
  1. 将所有硬件事件记录到日志中
    1. 时钟中断，网络中断，I/O中断等。
    2. 对于非确定性指令，记录附加信息
      1.  例如，记录时间戳寄存器的值
      2. on replay：从日志返回值，而不是实际的寄存器
  1. Reply：在相同的指令下以相同的顺序传送输入
    1. 如果在记录期间传送时钟中断，则执行第n条指令
    2. 在重放期间还在第n个指令提供时钟中断
  1. 给定事件日志，确定性重放重新创建VM
    1. 管理程序提供第一个事件
    2. 让机器执行到下一个事件
      - 使用特殊的硬件寄存器停止处理器在正确的指令
    3. 虚拟x86在重放期间与在记录期间相同地执行
      1. OS运行相同
      2. 应用程序运行相同
      3. ->在录音过程中，将在重放时产生相同的输出
  1. 限制：无法处理多核处理器x86
    - 记录和重放正确的指令交错太贵了

确定性重放对VM-FT的应用：
  1. 管理程序在 primary 记录
    - 通过日志通道将日志条目发送到备份
  2. 备份的管理程序重放日志条目
    1. 我们需要在下一个事件的指令处停止虚拟x86
    2. 我们需要什么下一个事件是
    3. ->备份滞后一个事件

例：
  1. primary接收网络中断
    1. 管理程序将中断和数据转发到备份
    2. 管理程序向操作系统内核提供网络中断
    3. OS内核运行
    4. 内核将数据包传送到服务器
    5. 服务器/内核对网卡写响应
    6. 管理程序获得控制并将响应放在电线上
  1. backup接收日志条目
    1. 备份提供网络中断
    2. 管理程序向其操作系统内核提供中断
    3. ... ...
    4. 服务器/内核向网卡发送响应
    5. 管理程序获得控制
      *不*在电线上放入响应
    6. 管理程序忽略本地时钟中断
      - 它从 primary 获得时钟中断
  1. 主和备份获得相同的输入，最终处于相同的状态

挑战1解决方案：FT协议
  1. primary延迟任何输出，直到备份ack
    1. 每个输出操作的日志条目
    2. primary在备份接收输出操作后发送输出
  1. 性能优化：
    1. primary 保持执行通过的输出操作
    2. 缓冲区输出，直到备份确认

Q：为什么要将输出事件发送到备份和延迟输出，直到备份已经确认？
  1. 考虑：不记录输出事件。仅记录输入事件。
  1. primary：
    1. 过程网络输入
    2. 产生输出
    3. primary 失败
  1. 备份无法正确再现此序列：
    1. 最后日志条目是：进程网络输入
      - 提交给内核
    2. 备份变成“活跃状态”
      - 它成为 primary
    3. 硬件中断（例如，时钟）
      - 交付给内核（因为备份现在是活跃的）
    4. 网络输入结果输出
      1. 在硬件中断之前或之后的 primary 有产生输出吗？
      1. 备份不知道
        - 它没有中断的日志条目
	1. 它没有输出的日志条目
      - 重要，因为时钟中断可能影响输出
  2. 通过发送开始输出事件到备份，备份可以正确地排序事件
    - 时钟中断在启动输出事件之前或之后

Q：primary和备份可以产生相同的输出事件吗？
  A：是的！
  1. primary 发送开始输出事件到备份
  2. primary 产生输出
  3. primary 失败
  4. 备份声明 primary 死
  5. 备份通过开始输出事件重播
  6. 备份变为活跃状态
  7. 备份执行输出事件
  8. ->作者声称产生输出两次是*不是一个问题
    1. 如果输出是网络包，客户端必须能够处理重复的包
    2. 如果输出写入磁盘，则将相同的数据写入同一位置两次
      1. 但是在中间不能有任何其他写入（它们将在日志中）
      2. 所以，应该确定
    3. 如果输出读取到磁盘，则将相同的数据读取两次到内存中的相同位置
      1. 使用反弹缓冲区来消除DMA竞争
      2. 传送 I/O 完成中断数据

问：当主节点在接收到网络输入之后但在发送相应的日志条目备份之前失败会发生什么？
  - A：网络输入。服务依赖客户端重试。
    - 这是合理的，因为网络可能丢失请求数据包
  - A：磁盘输入。虚拟机管理程序重新启动挂起的磁盘 I/O

挑战2解决方案：共享磁盘
  1. 困难问题只有不可靠的网络！
    - 参见下周的讲座
  1. Paper的解决方案：假设共享磁盘
    1. 备份通过最后一个日志条目重放
    2. 在磁盘上备份原子性测试和设置变量
      1. 如果设置，primary仍然活着。自杀
      2. 如果没有设置，primary就死了。成为primary
      3. 如果是主数据库，请从检查点创建新备份
        - 使用VMotion

1. 共享存储是单点故障
    - 如果共享存储已关闭，服务将关闭
  - 确定纸张的设置，但地理复制服务
    - 地震时无法生存

Q：为什么不在共享磁盘上有主记录日志，然后备份重播日志？

  优点：无需日志记录通道，无FT协议等。

Q：主服务器失败后，服务不可用多长时间？
  1. 检测故障
  1. 执行日志条目
    - 如果备份过于落后，VM-FT会降低主计算机
  1. 写入共享存储
  1. 切换到“live”

实施挑战：
  1. 网络堆栈
    1. 异步事件陷阱到管理程序
    2. 减少将网络输出传送到备份所造成的额外延迟
  2. 磁盘 I/O
    - DMA

性能（表1）
  1. FT /非FT：印象深刻！
    - 小慢下来
  1. 记录带宽
    - 18 Mbit / s的my-sql
      1. 到期，因为备份不会从磁盘读取
      2. 从locak磁盘读取：8 Mbit/s
  2. 网络应用：FT显着限制网络带宽
    - 所有输入包必须转发到备份
  3. 不够好的高性能服务？

概要：
  1. 主要备份复制
    - VM-FT：干净的例子
  2. 如何处理裂脑综合征？
    - 下一讲
  3. 如何获得更好的性能？
    - 使用更高级别的复制状态机的主 - 后复制 键/值操作，如put和get


----

VMware知识库（＃1013428）谈到多CPU支持。VM-FT可能已切换
从复制状态机方法到状态转移方法，但是
不清楚这是否是真实的。

http://www.wooditwork.com/2014/08/26/whats-new-vsphere-6-0-fault-tolerance/
http://www.tomsitpro.com/articles/vmware-vsphere-6-fault-tolerance-multi-cpu,1-2439.html
