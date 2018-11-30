smgr.c

#Data Structures
```
/*
 * smgr.c maintains a table of SMgrRelation objects, which are essentially
 * cached file handles.  An SMgrRelation is created (if not already present)
 * by smgropen(), and destroyed by smgrclose().  Note that neither of these
 * operations imply I/O, they just create or destroy a hashtable entry.
 * (But smgrclose() may release associated resources, such as OS-level file
 * descriptors.)
 *
 * An SMgrRelation may have an "owner", which is just a pointer to it from
 * somewhere else; smgr.c will clear this pointer if the SMgrRelation is
 * closed.  We use this to avoid dangling pointers from relcache to smgr
 * without having to make the smgr explicitly aware of relcache.  There
 * can't be more than one "owner" pointer per SMgrRelation, but that's
 * all we need.
 *
 * SMgrRelations that do not have an "owner" are considered to be transient,
 * and are deleted at end of transaction.
 */
typedef struct SMgrRelationData
{
	/* rnode is the hashtable lookup key, so it must be first! */
	RelFileNodeBackend smgr_rnode;		/* relation physical identifier */

	/* pointer to owning pointer, or NULL if none */
	struct SMgrRelationData **smgr_owner;

	/*
	 * These next three fields are not actually used or manipulated by smgr,
	 * except that they are reset to InvalidBlockNumber upon a cache flush
	 * event (in particular, upon truncation of the relation).  Higher levels
	 * store cached state here so that it will be reset when truncation
	 * happens.  In all three cases, InvalidBlockNumber means "unknown".
	 */
	BlockNumber smgr_targblock; /* current insertion target block */
	BlockNumber smgr_fsm_nblocks;		/* last known size of fsm fork */
	BlockNumber smgr_vm_nblocks;	/* last known size of vm fork */

	/* additional public fields may someday exist here */

	/*
	 * Fields below here are intended to be private to smgr.c and its
	 * submodules.  Do not touch them from elsewhere.
	 */
	int			smgr_which;		/* storage manager selector */

	/* for md.c; NULL for forks that are not open */
	struct _MdfdVec *md_fd[MAX_FORKNUM + 1];

	/* if unowned, list link in list of all unowned SMgrRelations */
	struct SMgrRelationData *next_unowned_reln;
} SMgrRelationData;

typedef SMgrRelationData *SMgrRelation;
```

#Functions
```
/* backend process init stage, called in BaseInit()*/
extern void smgrinit(void);
/*
 * Macro RelationOpenSmgr is used to open smgr to write blocks for a Relation
 * Here the owner mean the field in Relation struct.
 * typedef struct RelationData{...struct SMgrRelationData *rd_smgr;	cached file handle, or NULL ...}
 * smgropen also store the smgr item SMgrRelationData into Hash Map SMgrRelationHash
 */
extern SMgrRelation smgropen(RelFileNode rnode, BackendId backend);
extern void smgrsetowner(SMgrRelation *owner, SMgrRelation reln);
extern void smgrclearowner(SMgrRelation *owner, SMgrRelation reln);

/*
 * call mdclose() to close the file handle. Also clean the items in SMgrRelationHash hashmap.
 * smgrclosenode is used by e.g. RelationSetNewRelfilenode(), just an interface when there is 
 * no 'SMgrRelation reln' in context
 */
extern void smgrclose(SMgrRelation reln);
extern void smgrcloseall(void);
extern void smgrclosenode(RelFileNodeBackend rnode);

/* check physical file exist?*/
extern bool smgrexists(SMgrRelation reln, ForkNumber forknum);
/* First create dir, then create file.*/
extern void smgrcreate(SMgrRelation reln, ForkNumber forknum, bool isRedo);
extern void smgrcreate_ao(RelFileNodeBackend rnode, int32 segmentFileNum, bool isRedo);

/*
 * unlink file cannot be revert.
 * firstly mdclose(), DropRelFileNodesAllBuffers() and CacheInvalidateSmgr(), finaly mdunlink()
 * DropRelFileNodesAllBuffers() will compare RelFileNode with every buffer in Buffer Poll,
 * and call InvalidateBuffer() to the target, add the buffer to buffer pool freelist.
 * CacheInvalidateSmgr() ??need more spike??
 */
extern void smgrdounlink(SMgrRelation reln, bool isRedo, char relstorage);
extern void smgrdounlinkall(SMgrRelation *rels, int nrels, bool isRedo, char *relstorages);
extern void smgrdounlinkfork(SMgrRelation reln, ForkNumber forknum,
				bool isRedo, char relstorage);

/*
 * call mdextend() to add a new block on disk file.
 * firstly call _mdfd_getseg() to get the segmentno of file.
 * then calcualte the seek position of the segment file.
 * then call FileSeek and FileWrite to add a new block.
 * finally register_dirty_segment() for untemp table and non-skipfsync flag.
 * In non-revocery mode, call RememberFsyncRequest() to delay fync.
 * In recovery mode ??need more spike??
 */
extern void smgrextend(SMgrRelation reln, ForkNumber forknum,
				   BlockNumber blocknum, char *buffer, bool skipFsync);

/* prefetch disk, not used in gpdb yet.*/
extern void smgrprefetch(SMgrRelation reln, ForkNumber forknum,
					 BlockNumber blocknum);
/*
 * Find the right file and seek to the position to read and write.
 * Every read should equal to BLCKSZ, except in recovery mode or 
 * GUC zero_damaged_pages is on.
 * Write need to consider whether to call register_dirty_segment() to fsync async.
 */
extern void smgrread(SMgrRelation reln, ForkNumber forknum,
				 BlockNumber blocknum, char *buffer);
extern void smgrwrite(SMgrRelation reln, ForkNumber forknum,
				  BlockNumber blocknum, char *buffer, bool skipFsync);

/*
 * Firstly call DropRelFileNodeBuffers() to removes from the buffer pool all the 
 * pages of the specified relation fork that have block numbers >= firstDelBlock.
 * Here firstDelBlock is parameter nblocks. When truncate a table, nblocks = 0,
 * which means to delete all the blocks of this fork.
 * Then call CacheInvalidateSmgr() ??need more spike??
 * Finally call FileTruncate().
 * Question: how to remove additional segment files except the first one?
 */
extern void smgrtruncate(SMgrRelation reln, ForkNumber forknum,
					 BlockNumber nblocks);

/* return block number */
extern BlockNumber smgrnblocks(SMgrRelation reln, ForkNumber forknum);

/*
 * smgrimmedsync is called by before transaction commit. e.g. skip WAL.
 * while smgrsync is called by checkpointer.
 */
extern void smgrimmedsync(SMgrRelation reln, ForkNumber forknum);
extern void smgrsync(void);
extern void smgrpreckpt(void);
extern void smgrpostckpt(void);
extern void AtEOXact_SMgr(void);
```
