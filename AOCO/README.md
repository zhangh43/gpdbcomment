Describe Varblock in picture.

How many kinds of files a AO/CO table have and their usage?

When you delete 90% of AO/CO table, will visibility map become too large? How to handle it?

TID of a AO/CO table?


typedef struct TupleTableSlot
{
	NodeTag		type;
	int         PRIVATE_tts_flags;      /* TTS_xxx flags */

	/* Heap tuple stuff */
	HeapTuple PRIVATE_tts_heaptuple;
	void    *PRIVATE_tts_htup_buf;
	uint32   PRIVATE_tts_htup_buf_len;

	/* Mem tuple stuff */
	MemTuple PRIVATE_tts_memtuple;
	void 	*PRIVATE_tts_mtup_buf;
	uint32  PRIVATE_tts_mtup_buf_len;
	ItemPointerData PRIVATE_tts_synthetic_ctid;	/* needed if memtuple is stored on disk */
	
	/* Virtual tuple stuff */
	int 	PRIVATE_tts_nvalid;		/* number of valid virtual tup entries */
	long    PRIVATE_tts_off; 		/* hack to remember state of last decoding. */
	bool    PRIVATE_tts_slow; 		/* hack to remember state of last decoding. */
	Datum 	*PRIVATE_tts_values;		/* virtual tuple values */
	bool 	*PRIVATE_tts_isnull;		/* virtual tuple nulls */

	TupleDesc	tts_tupleDescriptor;	/* slot's tuple descriptor */
	MemTupleBinding *tts_mt_bind;		/* mem tuple's binding */ 
	MemoryContext 	tts_mcxt;		/* slot itself is in this context */
	Buffer		tts_buffer;		/* tuple's buffer, or InvalidBuffer */

    /* System attributes */
    Oid         tts_tableOid;
} TupleTableSlot;




/*
 * used for scan of append only relations using BufferedRead and VarBlocks
 */
typedef struct AppendOnlyScanDescData
{
	/* scan parameters */
	Relation	aos_rd;				/* target relation descriptor */
	Snapshot	appendOnlyMetaDataSnapshot;

	/*
	 * Snapshot to use for non-metadata operations.
	 * Usually snapshot = appendOnlyMetaDataSnapshot, but they
	 * differ e.g. if gp_select_invisible is set.
	 */ 
	Snapshot    snapshot;

	Index       aos_scanrelid;
	int			aos_nkeys;			/* number of scan keys */
	ScanKey		aos_key;			/* array of scan key descriptors */
	
	/* file segment scan state */
	int			aos_filenamepath_maxlen;
	char		*aos_filenamepath;
									/* the current segment file pathname. */
	int			aos_total_segfiles;	/* the relation file segment number */
	int			aos_segfiles_processed; /* num of segfiles already processed */
	FileSegInfo **aos_segfile_arr;	/* array of all segfiles information */
  /*是否需要去读新的segfile，当关闭一个已读完文件或者*/
	bool		aos_need_new_segfile;
  /*
   *与bufferDone对应，aos_done_all_segfiles表示一个ao表是否读完，
   *可通过aos_total_segfiles 和 aos_segfiles_processed来确定 
   */
	bool		aos_done_all_segfiles;
	
	MemoryContext	aoScanInitContext; /* mem context at init time */

	int32			usableBlockSize;
	int32			maxDataLen;

	AppendOnlyExecutorReadBlock	executorReadBlock;

  /*
   * current scan state
   * bufferDone是指当前varblock是否读完。
   */
	bool		bufferDone;

	bool	initedStorageRoutines;

	AppendOnlyStorageAttributes	storageAttributes;
	AppendOnlyStorageRead		storageRead;

	char						*title;
				/*
				 * A phrase that better describes the purpose of the this open.
				 *
				 * We manage the storage for this.
				 */
	
	/*
	 * The block directory info.
	 *
	 * For AO tables, the block directory is built during the first index
	 * creation. If set indicates whether to build block directory while
	 * scanning.
	 */
	AppendOnlyBlockDirectory *blockDirectory;

	/**
	 * The visibility map is used during scans
	 * to check tuple visibility using visi map.
	 */ 
	AppendOnlyVisimap visibilityMap;

}	AppendOnlyScanDescData;

typedef AppendOnlyScanDescData *AppendOnlyScanDesc;







/*
 * Descriptor of a single AO relation file segment.
 */
typedef struct FileSegInfo
{
	TupleVisibilitySummary tupleVisibilitySummary;

	/* the file segment number */
	int			segno;

	/* total (i.e. including invisible) number of tuples in this fileseg */
	int64		total_tupcount;

	/* total number of varblocks in this fileseg */
	int64		varblockcount;

	/*
	 * Number of data modification operations
	 */
	int64		modcount;

	/* the effective eof for this segno  */
	int64		eof;

	/*
	 * what would have been the eof if we didn't compress this rel (= eof if
	 * no compression)
	 */
	int64		eof_uncompressed;

	/* File format version number */
	int16		formatversion;

	/*
	 * state of the segno. The state is only maintained on the segments.
	 */
	FileSegInfoState state;

} FileSegInfo;



什么是varblock
Heap表中block／page


typedef struct BufferedAppend
{
	/*
	 * Init level.
	 */
	char				*relationName;

	/*
	 * Large-write memory level members.
	 */
    int32      			 maxBufferLen;
    int32      			 maxLargeWriteLen;

	uint8                *memory;
    int32                memoryLen;

    uint8                *largeWriteMemory;

    uint8                *afterBufferMemory;
							/*
							 * We allocate maxBufferLen bytes after largeWriteMemory
							 * to support buffers that cross large write boundaries.
							 */
	
	int64				 largeWritePosition;
    int32                largeWriteLen;
							/*
							 * The position within the current file for the next write
							 * and the number of bytes buffered up in largeWriteMemory
							 * for the next write.
							 *
							 * NOTE: The currently allocated buffer (see bufferLen)
							 * may spill across into afterBufferMemory.
							 */
	
	/*
	 * Buffer level members.
	 */	
    int32                bufferLen;
							/*
							 * The buffer within largeWriteMemory given to the caller to
							 * fill in.  It starts after largeWriteLen bytes (i.e. the
							 * offset) and is bufferLen bytes long.
							 */

	/*
	 * File level members.
	 */
	File 				 file;
	RelFileNodeBackend	relFileNode;
	int32				segmentFileNum;
    char				 *filePathName;
    int64                fileLen;
    int64				 fileLen_uncompressed; /* for calculating compress ratio */

} BufferedAppend;

这个结构体负责高效将内存buffer写入AO的segfile文件。largeWriteMemory是需要落盘的memory地址， largeWriteLen是需要落盘的长度。
largeWritePosition 记录文件的当前有效位置（offset）。 

AO table 首先写入varblock， 写满之后调用AppendOnlyStorageWrite_Content

AO表文件结构，
AO表Scan：
1. SeqScan
2. BitmapScan  
AO表Insert
AO表的insert是append模式，只能从eof开始不对追加数据。因此，无法处理多个tx并发insert同一个文件。GPDB的处理方式是提前为每一个tx分配关联的
segfileno，不同的tx插入不同的segfile。具体来说ExectorStart的InitPlan阶段QD调用assignPerRelSegno获取每一个result relation当前可用的segfileno；QE从QD获取segfileno。
segfileno，不同的tx插入不同的segfile。具体来说ExectorStart的InitPlan阶段QD调用assignPerRelSegno获取每一个result relation当前可用的segfileno；QE从QD获取segfileno。
Insert初始化阶段，主要是准备AppendOnlyInsertDescData结构体，包括：设置待插入的segfileno，是否使用toast，从appendonly系统表读取blocksize，storage
属性信息（压缩类型，压缩等级，checksum，safeFSWriteSize，overflowSize；补充解释后面两个属性：safeFSWriteSize是在不成熟的文件系统上，根据其最小的安全写入大小（避免数据损坏），在gpdb会做额外的padding处理. overflowSize是指压缩算法在针对某些数据时，压缩后的大小甚至会超过数据原始大小，超过的原数据大小的部分就是overflowSize，比如GPDB的quicklz算法的overflowSize是400bytes）初始化AppendOnlyStorageWrite结构体，主要包括未压缩数据buffer和长度，uncompressedBuffer／maxBufferLen 压缩数据后的最大长度maxBufferWithCompressionOverrrunLen（maxBufferLen+compressionOverrunLen）其实compressionOverrunLen就是overflowSize。 largeWriteLen是两倍的maxBufferLen。 BufferedAppend结构体， completeHeaderLen包括2*checksum，firstrownum +（AoHeader_RegularSize／AoHeader_LongSize）。  tempSpace／tempSpaceLen用于varblock缓存 tuple是否toast化的阈值toast_tuple_threshold。 varblockCount当前tx插入varblock的数量，作为增量在tx提交时写入pg_aoseg.pg_aoseg_oid 系统表。bufferCount 当前varblock个数，不是增量，是总数。insertCount当前tx插入的tuple数，同样增量。 SetCurrentFileSegForWrite（） 函数负责对待插入文件，加锁，获取pg_aoseg.pg_aoseg_oid 系统表的信息，并打开文件描述符以备使用。
AO表Update+Delete


