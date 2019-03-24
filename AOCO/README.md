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
