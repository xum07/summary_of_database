本质来说，redo日志的功能是：所有对数据库的修改动作都会记录一份日志到磁盘中，防止数据库系统因为异常（断电、节点间网络断连等）而出现数据没能及时写到磁盘中，导致数据。有了这份日志，就可以根据日志中的记录信息，恢复数据。

上面这段话潜含了几个信息：

1. 对数据库的修改动作是否生效是以日志为准的。换句话说，如果日志也因为异常没能够保存到磁盘上，那这个操作涉及的数据丢了也无计可施
2. 数据库的机制是日志先行，即所谓的WAL(Write Ahead Log)：预写式日志
3. redo日志就是用来做数据恢复（或者也称为回放）用的

达成上面的共识之后，接下来详细看下PosrgreSQL(以下简称PG)的redo log的细节

# 初始化

在PG执行完`initdb`之后，会在`$PGDATA/pg_wal`目录下生成一份日志文件和一个用来保存归档日志的目录

![2022-03-23-20-37-07-image.png](./redo_log.assets/2022-03-23-20-37-07-image.png)

在`initdb`执行期间，redo_log被初始化的流程为：

```c
initdb
  ├── create_xlog_or_symlink  // 创建redo_log的目录
  └── bootstrap_template1
           └── 调用管道执行：postgres --boot -x1
                   └── main
                         └── AuxiliaryProcessMain
                                 └── BootStrapXLOG
                                        └── XLogFileInit(1, &use_existent, false)
```

其中，redo_log的文件名由`XLogFilePath`得出，在代码中的详细定义如下，可以计算得到在`BootStrapXLOG`调用期间，各项参数为`XLogFilePath(path, 1, 1, 16*1024*1024)`，最终计算得到的文件名为：`00000001-00000000-00000001`

> 最大的redo_log文件名是：`00000001-FFFFFFFF-000000FF`，因为`XLogSegmentsPerXLogId`的最大值是`FF`

```c
#define XLogFilePath(path, tli, logSegNo, wal_segsz_bytes)	\
	snprintf(path, MAXPGPATH, XLOGDIR "/%08X%08X%08X", tli,	\
			 (uint32) ((logSegNo) / XLogSegmentsPerXLogId(wal_segsz_bytes)), \
			 (uint32) ((logSegNo) % XLogSegmentsPerXLogId(wal_segsz_bytes)))

#define XLOGDIR	"pg_wal"

#define XLogSegmentsPerXLogId(wal_segsz_bytes) (UINT64CONST(0x100000000) / (wal_segsz_bytes))

int	wal_segment_size = DEFAULT_XLOG_SEG_SIZE;
#define DEFAULT_XLOG_SEG_SIZE	(16*1024*1024)
```

> `wal_segment_size`参数除了上述代码中给定的默认值外：
>
> 1. 在PG 10版本时是放在configure执行，通过`--with-wal-segsize`参数来指定的，一旦系统启动后就无法更改
>
> 2. 在PG 更高的版本时(没有细查引入该修改版本，仅知道PG 14中已经是了)，`--with-wal-segsize`参数从configure中移除，转而放在了`pg_resetwal`中，变为了`--wal-segsize`参数
>
>    ![image-20220323213152532](redo_log.assets/image-20220323213152532.png)

此时创建的redo_log中除了一个文件头，并没有什么有效数据，只以全0填充，最终的大小是等于`wal_segment_size=16777216`

# 文件结构

对redo_log而言，其文件结构如下：

![image-20220323223120677](redo_log.assets/image-20220323223120677.png)

1. 在每个segment的第一个page的头部，会写入`XLogLongPageHeaderData`

   > redo_log的blocksize是通过configure根据参数`--with-wal-blocksize`在启动前就设置好的，默认为8K，与PG系统中所有的page页面大小保持一致

   ```c
   typedef struct XLogLongPageHeaderData
   {
   	XLogPageHeaderData std;		/* standard header fields */
   	uint64		xlp_sysid;		/* 根据pg_control得到，本质是由系统时间计算得出 */
   	uint32		xlp_seg_size;	/* just as a cross-check，即wal_segment_size，默认16M */
   	uint32		xlp_xlog_blcksz;	/* just as a cross-check，即blocksize，默认8K */
   } XLogLongPageHeaderData;
   
   gettimeofday(&tv, NULL);
   sysidentifier = ((uint64) tv.tv_sec) << 32;
   sysidentifier |= ((uint64) tv.tv_usec) << 12;
   sysidentifier |= getpid() & 0xFFF;
   longpage->xlp_sysid = sysidentifier;
   ```

2. 在每个page的头部(不是segment的第一个page时)，会写入一个`XLogPageHeaderData`

   ```
   /*
    * Each page of XLOG file has a header like this:
    */
   #define XLOG_PAGE_MAGIC 0xD10D	/* can be used as WAL version indicator */
   
   typedef struct XLogPageHeaderData
   {
   	uint16		xlp_magic;		/* magic value for correctness checks，固定为XLOG_PAGE_MAGIC */
   	uint16		xlp_info;		/* flag bits, see below */
   	TimeLineID	xlp_tli;		/* TimeLineID of first record on page */
   	XLogRecPtr	xlp_pageaddr;	/* XLOG address of this page */
   
   	/*
   	 * When there is not enough space on current page for whole record, we
   	 * continue on the next page.  xlp_rem_len is the number of bytes
   	 * remaining from a previous page; it tracks xl_tot_len in the initial
   	 * header.  Note that the continuation data isn't necessarily aligned.
   	 */
   	uint32		xlp_rem_len;	/* total len of remaining data for record */
   } XLogPageHeaderData;
   ```

3. 除开文件头的部分，一条真正的redo_log记录部分由`XLogRecord`+数据部分组成。其中，`XLogRecord`如下，而数据部分对于不同类型的操作，其实际存放的信息有所不同。在PG 9.5版本之后，数据部分的结构如下

   ```c
   typedef struct XLogRecord
   {
   	uint32		xl_tot_len;		/* total len of entire record */
   	TransactionId xl_xid;		/* xact id */
   	XLogRecPtr	xl_prev;		/* ptr to previous record in log */
   	uint8		xl_info;		/* flag bits, see below */
   	RmgrId		xl_rmid;		/* resource manager for this record */
   	/* 2 bytes of padding here, initialize to zero */
   	pg_crc32c	xl_crc;			/* CRC for this record */
   	/* 后面紧接着的是XLogRecordBlockHeaders或XLogRecordDataHeader */
   } XLogRecord;
   ```

   - 例如对于checkpoint记录而言，其redo_log记录为

     ![image-20220323223907457](redo_log.assets/image-20220323223907457.png)

   - 例如对insert一条数据而言，其redo_log记录为

     ![image-20220323224007039](redo_log.assets/image-20220323224007039.png)

     > 其中，`xl_heap_header`和`Tuple B data`直接对应tuple的header和data
     >
     > ![image-20220323224624152](redo_log.assets/image-20220323224624152.png)

## 哪些操作会记入redo_log

这一点其实涉及到`XLogRecord`结构中的`xl_rmid`和`xl_info`，`rmid`全称是Resource Manager Identity(资源管理器Id)。`xl_rmid`和`xl_info`一起构成了一个唯一操作，所有会被记录到redo_log中的操作都会有这两个信息

> 在PG 14版本中，`xl_rmid`多了一个`_ID`后缀（不清楚具体哪个版本引入的），至少在PG 10版本中还没有

- 如对于checkpoint关闭，记录的是`xl_rmid = RM_XLOG_ID, xl_info = XLOG_CHECKPOINT_SHUTDOWN`
- 如对xlog自身新增page，记录的是`xl_rmid = RM_XLOG_ID, xl_info = XLOG_FPI`

可以根据`xl_rmid`对操作的类别进行划分如下：

| 操作           | 资源管理器Id                                               |
| -------------- | ---------------------------------------------------------- |
| 堆表操作       | RM_HEAP, RM_HEAP2                                          |
| 索引操作       | RM_BTREE, RM_HASH, RM_GIN, RM_GIST, RM_SPGIST, RM_BRIN     |
| 序列操作       | RM_SEQ                                                     |
| 事务操作       | RM_XACT, RM_MULTIXACT, RM_CLOG, RM_XLOG, RM_COMMIT_TS      |
| 表空间操作     | RM_SMGR, RM_DBASE, RM_TBLSPC, RM_RELMAP                    |
| 副本或备机操作 | RM_STANDBY, RM_REPLORIGIN, RM_GENERIC_ID, RM_LOGICALMSG_ID |

# 写入一条redo_log记录

Full Page Write

# 内存管理

redo_log在PG拉起时会分配一定的内存空间，其流程为：

```c
XLOGShmemInit
```



# 落盘



# 参考资料

1. [Write Ahead Logging — WAL](https://www.interdb.jp/pg/pgsql09.html)
2. [PostgreSQL中的XLOG(一)基础概念和初始化](http://www.postgres.cn/news/viewone/1/334)
3. [full page write 机制](http://mysql.taobao.org/monthly/2015/11/05/)