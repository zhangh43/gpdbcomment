Questions:

1. How does PG handle CPU cache missing and Memory barrier 
For cache missing/coherence, using volatile keyword.

For memory barrier, there are two types of barriers:

pg_memory_barrier()
pg_memory_barrier() can always substitute for either a read or a write barrier, but is typically more expensive, and
therefore should be used only when needed.

pg_read_barrier() and pg_write_barrier()
When a memory barrier is being used to separate two loads, use pg_read_barrier(); when it is separating two stores, 
use pg_write_barrier(); when it is a separating a load and a store (in either order), use pg_memory_barrier().

2. How to choose the right lock to use.
Spin locks, in particular, are typically only held for a handful of machine instructions, so a PostgreSQL process stuck on a spinlock would almost certainly indicate an outright bug.  But because the code that gets executed while holding a spinlock is typically extremely simple, such bugs rarely occur in practice

Lightweight locks can be a little more problematic.  If you see a process that's definitely stopped (it's not consuming any CPU or I/O resources) and not waiting for a heavyweight lock, chances are good it's waiting for a lightweight lock.  Unfortunately, PostgreSQL does not have great tools for diagnosing such problems; we probably need to work a little harder in this area.  The most typical problem with lightweight locks, however, is not that a process acquires one and sits on it, blocking everyone else, but that the lock is heavily contended and there are a series of lockers who slow each other down.  Forward progress continues, but throughput is reduced.  Troubleshooting these situations is also difficult.  One common case that is often easily fixed is contention on WALWriteLock, which can occur if the value of wal_buffers is too small, and can be remedied just by boosting the value.

3. Which lock will be released when transaction abort automatically?

lw When transaction abort auto release
codes holding spin lock should not elog error.


4. Describe the usage of different lock mode
./src/include/storage/lock.h
```
/* NoLock is not a lock mode, but a flag value meaning "don't get a lock" */
#define NoLock                  0

#define AccessShareLock         1       /* SELECT */
#define RowShareLock            2       /* SELECT FOR UPDATE/FOR SHARE */
#define RowExclusiveLock        3       /* INSERT, UPDATE, DELETE */
#define ShareUpdateExclusiveLock 4      /* VACUUM (non-FULL),ANALYZE, CREATE
                                         * INDEX CONCURRENTLY */
#define ShareLock               5       /* CREATE INDEX (WITHOUT CONCURRENTLY) */
#define ShareRowExclusiveLock   6       /* like EXCLUSIVE MODE, but allows ROW
                                         * SHARE */
#define ExclusiveLock           7       /* blocks ROW SHARE/SELECT...FOR
                                         * UPDATE */
#define AccessExclusiveLock     8       /* ALTER TABLE, DROP TABLE, VACUUM
                                         * FULL, and unqualified LOCK TABLE */
```


5 Lock conflict table?
https://www.postgresql.org/docs/current/explicit-locking.html


6 why use session level lock?
ref index_drop. using two transactions to do index drop
Step1:??
Step2:??

/*
 *		LockRelationIdForSession
 *
 * This routine grabs a session-level lock on the target relation.  The
 * session lock persists across transaction boundaries.  It will be removed
 * when UnlockRelationIdForSession() is called, or if an ereport(ERROR) occurs,
 * or if the backend exits.
 *
 * Note that one should also grab a transaction-level lock on the rel
 * in any transaction that actually uses the rel, to ensure that the
 * relcache entry is up to date.
 */
void
LockRelationIdForSession(LockRelId *relid, LOCKMODE lockmode)


7 How to implement deadlock detector?
Using timeout mechenism. 
RegisterTimeout(DEADLOCK_TIMEOUT, CheckDeadLockAlert); 
Details in timeout.c

8 What lockgroup?

9 What is ProcLock?

10 How to store the lock held info, wait info.

11 What is struct LockTag?
It describe the lockable object, could be a relation, tuple, page etc.
```
typedef enum LockTagType
{
	LOCKTAG_RELATION,			/* whole relation */
	/* ID info for a relation is DB OID + REL OID; DB OID = 0 if shared */
	LOCKTAG_RELATION_EXTEND,	/* the right to extend a relation */
	/* same ID info as RELATION */
	LOCKTAG_PAGE,				/* one page of a relation */
	/* ID info for a page is RELATION info + BlockNumber */
	LOCKTAG_TUPLE,				/* one physical tuple */
	/* ID info for a tuple is PAGE info + OffsetNumber */
	LOCKTAG_TRANSACTION,		/* transaction (for waiting for xact done) */
	/* ID info for a transaction is its TransactionId */
	LOCKTAG_VIRTUALTRANSACTION, /* virtual transaction (ditto) */
	/* ID info for a virtual transaction is its VirtualTransactionId */
	LOCKTAG_SPECULATIVE_TOKEN,	/* speculative insertion Xid and token */
	/* ID info for a transaction is its TransactionId */
	LOCKTAG_OBJECT,				/* non-relation database object */
	/* ID info for an object is DB OID + CLASS OID + OBJECT OID + SUBID */

	/*
	 * Note: object ID has same representation as in pg_depend and
	 * pg_description, but notice that we are constraining SUBID to 16 bits.
	 * Also, we use DB OID = 0 for shared objects such as tablespaces.
	 */
	LOCKTAG_USERLOCK,			/* reserved for old contrib/userlock code */
	LOCKTAG_ADVISORY			/* advisory user locks */
} LockTagType;
```
12 what is LockMethodData
LockMethodData could be considered as a deescription of a lock method/system. How many locks it supported and the conflict map between these locks. 
```
typedef struct LockMethodData
{
	int			numLockModes;
	const LOCKMASK *conflictTab;
	const char *const *lockModeNames;
	const bool *trace_flag;
} LockMethodData;
```
There are two LockMethodData in Postgres. 
```
/* These identify the known lock methods */
#define DEFAULT_LOCKMETHOD	1
#define USER_LOCKMETHOD		2
```
Take default lockmethod as example.
```
const LockMethodData default_lockmethod = {
	AccessExclusiveLock,		/* highest valid lock mode number */
	LockConflicts,
	lock_mode_names,
#ifdef LOCK_DEBUG
	&Trace_locks
#else
	&Dummy_trace
#endif
};
```
LockConflicts is an int array, record the conflict map
```
static const LOCKMASK LockConflicts[] = {
	0,

	/* AccessShareLock */
	(1 << AccessExclusiveLock),

	/* RowShareLock */
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock),

	/* RowExclusiveLock */
	(1 << ShareLock) | (1 << ShareRowExclusiveLock) |
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock),

	/* ShareUpdateExclusiveLock */
	(1 << ShareUpdateExclusiveLock) |
	(1 << ShareLock) | (1 << ShareRowExclusiveLock) |
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock),

	/* ShareLock */
	(1 << RowExclusiveLock) | (1 << ShareUpdateExclusiveLock) |
	(1 << ShareRowExclusiveLock) |
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock),

	/* ShareRowExclusiveLock */
	(1 << RowExclusiveLock) | (1 << ShareUpdateExclusiveLock) |
	(1 << ShareLock) | (1 << ShareRowExclusiveLock) |
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock),

	/* ExclusiveLock */
	(1 << RowShareLock) |
	(1 << RowExclusiveLock) | (1 << ShareUpdateExclusiveLock) |
	(1 << ShareLock) | (1 << ShareRowExclusiveLock) |
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock),

	/* AccessExclusiveLock */
	(1 << AccessShareLock) | (1 << RowShareLock) |
	(1 << RowExclusiveLock) | (1 << ShareUpdateExclusiveLock) |
	(1 << ShareLock) | (1 << ShareRowExclusiveLock) |
	(1 << ExclusiveLock) | (1 << AccessExclusiveLock)

};
```
lock_mode_names is just the name of eight locks
```
/* Names of lock modes, for debug printouts */
static const char *const lock_mode_names[] =
{
	"INVALID",
	"AccessShareLock",
	"RowShareLock",
	"RowExclusiveLock",
	"ShareUpdateExclusiveLock",
	"ShareLock",
	"ShareRowExclusiveLock",
	"ExclusiveLock",
	"AccessExclusiveLock"
};
```

13 what is struct LOCK
struct LOCK is the item in Shared memory hash table LockMethodLockHash;
Key is lockable object, value includes 
grantMask: record the lock type which already be granted. E.g. for a table ta, the AccessShareLock and RowShareLock may already be granted/held. This is used to check whether new AcquireLock request is allowed or not. Since AccessShareLock and RowShareLock are held, ExclusiveLock and AccessExclusiveLock will be denied.
waitMask: record the conflict lock type which is waiting to be allowed. If AcquireLock failed, the lock type will be written into waitMask
procLocks: PROCLOCK list contains backends, which are holding lock on this lockable object
waitProcs：PGPROC list contains backends which are waiting lock on this lockable object.
requested: request times for each lock type.
granted: grant times for each lock type.
```
typedef struct LOCK
{
	/* hash key */
	LOCKTAG		tag;			/* unique identifier of lockable object */

	/* data */
	LOCKMASK	grantMask;		/* bitmask for lock types already granted */
	LOCKMASK	waitMask;		/* bitmask for lock types awaited */
	SHM_QUEUE	procLocks;		/* list of PROCLOCK objects assoc. with lock */
	PROC_QUEUE	waitProcs;		/* list of PGPROC objects waiting on lock */
	int			requested[MAX_LOCKMODES];	/* counts of requested locks */
	int			nRequested;		/* total of requested[] array */
	int			granted[MAX_LOCKMODES]; /* counts of granted locks */
	int			nGranted;		/* total of granted[] array */
} LOCK;
```

