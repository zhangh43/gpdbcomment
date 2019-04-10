# AO Insert流程

1. 初始化阶段

读取append_only表，获取压缩（算法，等级），checksum，blocksize等信息
设置各种Buffer和长度信息，包括： overflowSize（压缩后可能增加的最大长度） 完整header长度（2个checksum+fisrtrownum+header—size），最大数据长度
填充storageWrite和BufferedWrite结构体，特别是初始化BufferedWrite的memory buffer
设置varblock的tempspace
设置toast阈值
加锁文件
如果不存在则创建新文件
打开文件
读取pg_aoseg.pg_aoseg_oid系统表，获取文件eof，varblock数量，tuple数量等信息。
（获取序列号）
准备一个新的varblock，供写入tuple
初始化索引blockdirectory

2. 插入tuple阶段

AO表插入时的输入参数有两个一个是tuple本身，一个是tupleoid。
Tuple本身是个memtuple，即tuple数据直接读取／写入磁盘。
tupleoid只针对has_oid类型的表。
开始插入时，首先判断是否需要进行toast处理，这里有两类条件，一个由GUCdebug_appendonly_use_no_toast控制useNoToast flag，一个由toast的阈值控制+是否包含Exernal属性决定，如果tuple大小超过toast阈值或者存在External属性，则进行toast处理。
AO的toast处理逻辑与Heap表没有本质区别，唯一不同是使用Memtuple作为toast处理对象。
如果表是has_oid类型，需要额外为其设置tupleoid。
之后进入插入的常规逻辑。首先根据tuple大小，尝试直接将buffer写入Varblock。以下两种条件会写入失败：1.当前Varblock的tuple数超过AOSmallContentHeader_MaxRowCount，2. 当前Varblock剩余的长度小于tuple的大小。对于上述情况会首先关闭当前非空varblock，并打开一个新的Varblock继续尝试写入，如果仍然失败则检查useNoToast flag，如果开启那么进入LargeContent写入模式，否则报错（应该使用toast处理超过一个varblock大小的数据）
LargeContent是一个tuple对应多个varblock的写入模式，首先生成LargeContentHeader，并输入buffer；之后开始smallcontent+data方式，不断将数据添加到一个个varblock中并写入磁盘。写入完毕打开一个新的varblock供接下来的tuple使用。我们可以看到一个Largecontent的数据插入最后修改tuple数等统计信息，并检查更新Sequence，设置aotupleid
3. 插入结束阶段

aoInsertDesc->usableBlockSize：ao blocksize
aoInsertDesc->completeHeaderLen：ao header长度，包括两个checksum，firstrow等信息
aoInsertDesc->maxDataLen: 一个aoblock的最大数据长度，aoInsertDesc->usableBlockSize -aoInsertDesc->completeHeaderLen;

storageWrite->maxBufferLen: ao blocksize
storageWrite->largeWriteLen(maxLargeWriteLen) = 2倍ao blocksize；2 * storageWrite->maxBufferLen
storageWrite->regularHeaderLen：不包括可能的AoHeader_LongSize 和 firstrow的header长度
storageWrite->logicalBlockStartOffset: 用于blockdirectory索引，当前数据写入后在文件中的偏移量，等于bufferedAppend->largeWritePosition+bufferedAppend->largeWriteLen， 即已经写入文件的位置，将内存中尚需写入的位置。
storageWrite->currentCompleteHeaderLen: 当前header完整长度
storageWrite->currentBuffer：指向当前bufferedAppend->largeWriteMemory
storageWrite->compressionOverrunLen:压缩算法引入的额外空间开销
storageWrite->maxBufferWithCompressionOverrrunLen：ao blocksize+ 压缩算法引入的开销。

bufferedAppend->bufferLen：记录当前buffer通过BufferedAppendGetBuffer获取的可供修改的数据长度。
bufferedAppend->largeWritePosition：写入文件时的文件偏移位置。一开始时eof，而后随着写入不断向后增长。
bufferedAppend->largeWriteLen当前buffer已经写入的数据长度，如果超过maxLargeWriteLen则将buffer largeWriteMemory写入磁盘.
bufferedAppend->largeWriteMemory:  待写入磁盘的buffer数据起点的指针，就是memory的位置
bufferedAppend->afterBufferMemory：从memory位置开始，bufferedAppend->maxLargeWriteLen之后的buffer指针。
bufferedAppend->maxBufferLen(maxBufferWithCompressionOverrrunLen)：包括压缩后产生oversize的最大可能buffer长度 is storageWrite->maxBufferWithCompressionOverrrunLen
bufferedAppend->maxLargeWriteLen: storageWrite->largeWriteLen = 2* blocksize
bufferedAppend->memoryLen:storageWrite->largeWriteLen+storageWrite->maxBufferWithCompressionOverrrunLen
bufferedAppend->memory: palloc, 没有显示释放，随上层memorycontext




# Update and delete
1.AO表的delete通过visimap实现，而update是由delete+insert实现。因此visimap是理解AO表的关键。

visimap是
与存储AO表eof信息的辅助heap表“pg_aoseg.pg_aoseg_<oid>”类似，visimap也被定义为一张辅助heap表“pg_aoseg.pg_aovisimap_<oid>”，它的catalog结构如下：
```
postgres=# create table a(i int) with (appendonly='true', orientation='row');
postgres=# insert into a select generate_series(1,1000);
postgres=# delete from a where i<=40;
DELETE 38
postgres=# select gp_segment_id,* from gp_dist_random('pg_aoseg.pg_aovisimap_248926');
 gp_segment_id | segno | first_row_no |        visimap
---------------+-------+--------------+------------------------
             1 |     1 |            0 | \x010000000001fe1f0000
             0 |     1 |            0 | \x010000000001feff0000
             2 |     1 |            0 | \x010000000001fe3f0000
(3 rows)
```

visimap的接口主要包括：
1. AppendOnlyVisimap_IsVisible： 判断一个AOTupleId是否可见, 主要用于AO表的scan（seq scan／bitmapAppendonlyscan），vacuum（压缩）操作
2. AppendOnlyVisimapDelete_Hide： 将一个AOTupleId隐藏，等效于删除操作，主要用于AO表的update，delete操作
3. AppendOnlyVisimap_GetSegmentFileHiddenTupleCount： 获取segfile的隐藏tuple数，用于在vacuum时判断是否需要压缩segfile。对于full vacuum只要
存在隐藏tuple，就进行压缩。对于vacuum操作，基于GUC gp_appendonly_compaction_threshold（默认值10%）决定是否进行压缩。

visimap的内部结构：
AppendOnlyVisimapEntry是一个visimap的最小单位，它用一个bitmap来表示tuple的可见性。AppendOnlyVisimapEntry通过属性segmentFileNum和
firstRowNum来定位bitmap的起始边界，基于起始边界我们可以快速确定一个AOtuple是否包含在当前AppendOnlyVisimapEntry中。
边界长度信息：未压缩的bitmap的最大size是4KB，最大tuple数是32768, 具体由以下宏来定义。
```
#define APPENDONLY_VISIMAP_MAX_RANGE 32768
#define APPENDONLY_VISIMAP_MAX_BITMAP_SIZE 4096
```

下面结合一个具体操作说明AppendOnlyVisimapEntry的作用
当判断一个tuple可见行的时候，实际上由两个步骤组成，第一步定位包含该tuple的bitmap（AppendOnlyVisimapEntry），第二步检查bitmap。

对于第一步，基于起始边界和边界长度，可以快速确定当前AppendOnlyVisimapEntry是否包含tuple，如果不包含，则需要额外替换和寻找目标AppendOnlyVisimapEntry。替换步骤需要将当前已修改（AppendOnlyVisimapEntry的dirty属性标识是否已修改）的AppendOnlyVisimapEntry落盘。落盘操作即将bitmap压缩并将当前AppendOnlyVisimapEntry形成tuple写入“pg_aoseg.pg_aovisimap_<oid>”heap表。寻找目标AppendOnlyVisimapEntry步骤主要是对“pg_aoseg.pg_aovisimap_<oid>”heap表进行indexscan，索引key就是segmentFileNum和firstRowNum。如果没有查找到，就初始化当前AppendOnlyVisimapEntry并将当前tuple作为该entry第一个tuple。

对于第二步，O（1）时间的bitmap位比较，这里还有个优化，如果bitmap为空，则之间返回tuple可见。

AppendOnlyVisimapDelete是支持AO表删除操作的辅助结构。它包括一个dirtyEntryCache和workfile。
下面结合一个具体操作说明AppendOnlyVisimapDelete的作用
AppendOnlyVisimapDelete是支持AO表删除操作的辅助结构。它包括属性dirtyEntryCache和workfile。
当删除（隐藏）一个tuple的时候，同样由两个步骤组成，第一步定位包含待删除tuple的AppendOnlyVisimapEntry，第二步置位AppendOnlyVisimapEntry的bitmap。
对于第一步看似和判断tuple可见性类似，但由于scan是只读操作，不会产生脏的AppendOnlyVisimapEntry需要落盘，因此定位的代价主要是对“pg_aoseg.pg_aovisimap_<oid>”heap表的indexscan。相反delete操作会产生大量的脏AppendOnlyVisimapEntry，而且批量数据的delete并不连续，会不断切换
AppendOnlyVisimapEntry， 如果每次切换都落盘性能会非常差。因此删除tuple时的定位操作使用了dirtyEntryCache和workfile相结合方式。当AppendOnlyVisimapEntry需要切换的时候，会首先将当前entry的原数据信息（workFileOffset，segmentFileNum和firstRowNum ）存入dirtyEntryCache，entry的bitmap信息压缩后存入workfile。在查找的时候，首先基于dirtyEntryCache查找，查找不到才通过“pg_aoseg.pg_aovisimap_<oid>”heap表索引查找。

对于第二步，同样O（1）时间的bitmap置位，但这里有三种情况。情况1是bitmap之前未置位，执行置位操作并返回HeapTupleMayBeUpdated。情况2bitmap已经置位且标记为脏，说明当前tx删除了该tuple，返回HeapTupleSelfUpdated；情况3bitmap已经置位且标记不为脏，有其他tx删除了该tuple，返回HeapTupleUpdated。最后无论哪种情况，都将该entry标记为脏。

AppendOnlyVisimapEntry， 

