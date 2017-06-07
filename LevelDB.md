# 参考文档
[原文](http://blog.csdn.net/tenfyguo/article/details/8454809)

# 整体架构
![](img/2017-06-07-00-14-58.png)

如图,主要包含六部分文件:
- 内存中:
    - MemTable
    - Immutable MemTable
- 磁盘中:
    - Current文件
    - Manifest文件
    - log文件
    - SSTable文件


当应用写入一条Key:Value数据时,LevelDB会先往log文件中写,成功后,再将记录插入到MEMTable中.这样基本就完成了写入操作.一次写入操作只涉及一次磁盘顺序写和一次内存写入,这就是LevelDB写入速度极快的原因.

Log文件的作用:防止系统崩溃而造成数据丢失.系统奔溃时.只需要将Log文件中的数据持久化到内存中即可.

当Memtable插入的数据占用内存到达一定界限后, 会将内存中的数据导出到磁盘中来进行持久化, LevelDB会生成新的MemTable和Log文件, 原来的MEMTable会转化为Immutable MemTable.
Immutable MemTable只读,不可进行写入和更改. 新数据会被记录到新的Log文件和新的MemTable中, 而后台调度会将Immutable MemTable的数据导出到磁盘, 形成一个新的SSTable文件(*.sst).

SSTable就是内存中的数据不断导出并进行Compaction操作后形成的. SSTable的所有文件是一种层级结构,第一层为level 0, 第二层为level 1等.
SSTable中的Key是有序的,文件中的数据按照Key的大小顺序进行排序. 各个层级的SSTable都是如此.但是level0的SSTable和其他层级的SSTable有所不同: 在level0的.sst文件中, 可能会存在Key重叠的情况,比如A中Key的范围{bar, car}, B中Key的范围{bar1, caa}, 很可能两个文件都存在Key='bar2'的数据. 这种现象在其他level的SSTable中不会出现.


Manifest文件用来记录SSTable各个文件的管理信息, 比如数据哪个level, 文件名叫啥, 最小Key和最大Key的值等,Manifest文件结构示意图:

![](img/2017-06-07-00-34-48.png)

Current文件: 记录当前Manifest文件名. 在LevelDB运行过程中, 随着Compaction的进行, SSTable文件结构会发生变化, 会有新的文件产生, 老的文件废弃, Manifest会反映出这种变化, 此时往往会生成新的Manifest来记载这种变化. Current的目的就是用来指出哪个Manifest文件才是我们需要关心的文件.

# log文件
log文件的布局:

![log文件布局](img/2017-06-07-22-27-00.png)

LevelDB将log文件切分为32K的Block, 每次读取以一个Block作为一个单位, 上图中一个log文件由三个Block组成, 从屋里布局上来看, log文件就是由多个连续的Block构成的. 

应用中是看不到这些Block的,只能看到Key:Value对, 在LevelDB中,将Key:Value对看做一条记录, 另外在这个数据前增加一个记录头, 用来记载一些管理信息.

记录头包含三个字段:
- CheckSum 是对"类型"和"数据"字段"的校验码, 当LevelDB读取数据时会对数据进行校验, 如果和CheckSum相同, 则说明数据完整, 没有遭到破坏. 
- "记录长度" 记载了数据的大小
- "数据" 则是上面将的Key:Value对.
- "类型"字段指出了每条记录的逻辑结构和log文件物理分块结构之间的关系, 具体而言,主要有以下四种类型:FULL/FIFST/MIDDLE/LAST.

如果记录类型是FULL, 代表了当前记录内容完整的存储在一个Block块中. 如果被分隔开,FIRST表示第一块数据LAST表示最后一块数据,中间的数据用MIDDLE表示.

LevelDB一次物理读取为一个Block, 然后将类型情况拼接处逻辑记录.

# SSTable文件
LevelDB中有不同层级的SSTable, SSTable将文件划分为固定大小的物理存储块, 内部数据按照Key的数据排序.  .sst文件结构入下图:

![](img/2017-06-07-22-42-00.png)

一个.sst文件被划分为固定大小的存储块, 每个Block分为三个部分:
- 数据存储Block,用来存储数据.
- Type区 用于标识数据存储区是否使用了数据加锁算法(Snappy算法)
- CRC 数据校验码, 用于判断数据在生成和传输时是否出错.

.sst文件的逻辑布局(每个数据块存储哪些内容, 包含哪些结构):

![](img/2017-06-07-22-45-54.png)

.sst文件被划分为两大区域:
- 数据存储区(Data Block) 用于存放实际的Key:Value数据
- 数据管理区 提供一些索引指针等管理数据, 目的是为了更快速便捷的查找相应的记录.

.sst文件的前面若干块实际存储KV数据, 后边数据管理区存储管理数据.

管理数据区又分为四种不同的类型:
- Meta Block
- Matablock Index(索引)
- Index block(数据索引块)
- Footer (文件尾部块)

LevelDB 1.2中对Meta Block并没有使用, 只保留了接口, 估计会在后续版本中加入内容.

看一下数据索引区(Index Block)的内部结构:

![](img/2017-06-07-22-51-03.png)

数据索引区是对Data Block中的数据建立的索引, 每条索引包含三个内容:
- 第一个字段记载大于等于数据块i中最大的Key值的那个Key
- 第二个字段指出数据块 i 在.sst文件中的起始位置
- 第三个字段指出Data Block i 的大小(有时候会有数据压缩)

索引里保存的Key未必是某条记录的Key. 上图中, Data Block i的索引Key的大小可以介于"world"和"www"之间,比如"worle". 也可以直接为"world".

Footer(文件末尾块)的内部结构:

metaindex_handle指出了Metablock Index的起始位置和大小

index_handle指出了Index Block的起始地址和大小.

这两个字段是为了正确读出索引的地址而设立的. 后面跟着一个填充区和魔数.

而数据存储区(Data Block)的数据部分内部布局如下图:

![](img/2017-06-07-23-07-06.png)

数据存储区内部也分为两部分: 前面是按照Key进行排序的数值记录, 在Block尾部则是有一些"重启点"(Restart Point), 指向Block内容中的一些记录的位置.

"重启点"的作用: 在Block内部Key是按照大小排序的, 相邻的两个Key很有可能Key存在部分重叠. 比如 key i="the car", key i+1="the color", Key i+1可以只存储olor部分, 两者的共同部分(the c)可以从Key i中获得. 通过重用Key的重复部分,减少存储开销. "重启点"的意思是: 从这条记录开始, 不在采用只记载不同Key部分, 而是重新记录整个Key值.

在Block内容区,数据结构是这样的:

![](img/2017-06-07-23-14-36.png)

包含五个字段:
- Key共享长度
- Key非共享长度
- Value长度
- Key非共享内容
- Value内容







