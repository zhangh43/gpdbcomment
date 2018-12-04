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

