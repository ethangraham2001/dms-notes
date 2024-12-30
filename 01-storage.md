# Memory And Storage

## Motivation and background

- Provide guarantees to the application. We want to make their lives easier.
- Big part of DBMS performance stems from proper implementation of this layer.
- Legacy designs focused on I/O optimization because it was slow.
- Complex hierarchy now with cloud etc... requirements are complex

## Memory hierarchy

### Thousand-foot view

- Provide illusion of very large memory, when scarce in reality.
- **Walls:** chip wall, memory wall, network wall
- Memory wall: never enough, and always slower than we want it to be
- Great big memory hierarchy, many variables involved (capacity, cost, 
latency, bandwidth)
- Different addressing (byte addressable in-memory random access vs block 
addressable sequential access on-disk)
- Huge performance gap between layers in memory / storage hierarchy
- Data movement is expensive in terms of energy and time
- Different units of transfer between different layers of memory hierarchy
- Managing hierarchy: improve temporal and spatial locality!
- NVM is rad, and so is CXL memory
- Disaggregated memory
- Cloud computing: forces separation between compute and storage
- Various storage systems: block storage, file storage, object storage
- Amazon elastic file system --> file storage paradigm, accessed as such by
the application.
- Object storage: Cloud-native stuff. Azure has different types of blobs.
- Amazon Redshift: database in the cloud.

### Second pass - summary

This section talks, at a high-level, about the problems related to memory and 
disk access. We like memory because it is fast, but it is scarce, and 
non-uniform in its access times due to a complex hierarchy. This variability 
makes it important to understand how memory is designed to optimize - remember, 
the goal is to make memory seem infinite despite physical constraints and 
non-uniformity.

Managing this memory hierarchy is non-trivial, and is getting harder with 
technological advancements such as NUMA, and cloud computing _(disaggregated
storage and compute, communication over network)_.

Various forms of cloud storage primitives exist - in the slides, **block 
storage, file storage, and object storage** are listed as examples. File 
storage most closely ressembles native storage accesses as it exposes the same 
API as a local file system. Cloud native platforms will often leverage various
types of caching to amortize the access times of network-attached-storage 
(NAS).

The problem of managing the memory and storage hierarchy is very old, but still
relevant as it is a battle that is still being fought.

## Segments and file storage

This section discusses how pages are stored at the physical layer, and the
various tradeoffs related to different decisions that can be w.r.t that.

- **RECALL:** want persistence, recovery, physical data independence --> 
big role in how DBMS chooses to store data
- Today: SSDs, NVM, NAS make things different than the old days of HDD - same
principles though
- DBMS does a lot of *things* at the same time, and each thing needs 
**logical** view of the data. Similar to how an OS handles stuff.
- **Tablespace:** logical unit in database (table, index, several tables...),
or something like a data structure for the engine (result buffer, undo buffer).
logical representation of the principal of spatial locality! Just an umbrella,
no continuity
- Segments, extents, blocks
- extents are contiguously allocated data blocks.
- DBMS blocks larger than OS page generally, and is smallest allocation unit
- Extents ensure logically continuous allocation of blocks - acts as the unit
of space allocation. Very easy to search sequentially, and can be dropped as
a unit.
- **Why extents?** it's a trade-off - DBMS optimize along many dimensions.
- DBMS offers tunable parameters *(everything is a tunable parameter)*
- Extent directory! We don't want too many entries *(see allocation 
strategies)*
- We don't want to allocate OS page by OS page when we are storing huge amounts
of data --> hence extents
- Slotted pages: has metadata at the top, and the tuples grow from the bottom
up. We keep a row dir and actual rows, as well as table dir *(schema)* and 
header (index, table, etc...). Can insert rows anywhere inside the block thanks
to the row directory. We can move rows around.
- Percentage free: reserve some space for updating tuples so that if we have to
perform an update, we don't have to shift data around between pages.
- Percentage used: how much space needs to be free for us to insert into the
block.
- we use a combination of percentage free and percentage used
- Note: a lot of DBMS map pages to blocks directly. So when we talk about 
`PCTFREE` and `PCTUSED`, these are the types of tunable parameters that may be
available to us with any arbitrary page layout that we opt for.
- Blocks suffer from fragmentation --> compaction very expensive
- Possible indirection when a tuple grows too big to fit inside of a block as 
we store a pointer
- Segments have free lists (not done at extent granularity)
- Tables, logically, are just unordered sets of tuples. It is important to
distinguish between logical implementation and physical storage (e.g. clustered
RIDs which is just a physical implementation detail, and not tied to the 
logical idea of a table).
- Shadow paging: when updating, copy page and do updates there. Throw away old
page. Great for recovery and concurrency control, however can destroy locality
if we have to move the new tuple to a new page. We get snapshot isolation!
- Should only use shadow paging when copying overhead is low. Maybe flash and
NVM.
- CoW: share a pointer for everyone, only make a copy for the modifying user
when they decide to modify it - other users still have the old version. Great 
for read-intensive workloads. 
- Delta File: copy the page into a special location, make updates into original
location. Great for undoing changes if necessary, and we can discard the delta
file when no longer needed.
- Keep old data in delta file: favours commits
- Keep new data in delta file: facours aborts
- Oracle used to use rollback segments (segments for storing old data copies)
and now use undo-tablespaces

Summary: IO is important, and there are a wide array of architectures and 
tradeoffs associated. The role that all of this plays in concurrency and data
access is not to be downplayed.

### Second pass - summary

This section discusses the persistent storage of DBMS pages - specifically how 
we handle the physical layout of the data such that we can provide the
following desirable properties while maintaining performance.

1. Persistence
2. Recoverability
3. Physical data independence

In particular, the slides make an example of Oracle 19, noting that despite it
being for slow HDD technology, most of key the insights hold true to today even
with substantial performance improvements along that dimension such as NVM.

The DBMS needs to have a logical view of the data so that physical resources
can be efficiently managed. Oracle 19, presented in the course, uses the 
following abstraction hierarchy, (where level `i` is one-to-many with `i+1`) 
providing us with a logical view of how the data is stored on disk.

1. Tablespaces _(logical unit of storage for well-defined entities such as 
tables, indexes, buffers)_
2. Segments _(set of extents)_
3. Extents _(regions of contiguously allocated blocks)_
4. Blocks _(maps one-to-many with OS file system blocks)_

The core idea here is that they are just _umbrellas_ that logically represent
the physical locality of data so that the DBMS can make informed decisions.

Extent management in particular is a good case study that demonstrates various
optimization decisions and trade-offs in DBMS.

As for blocks, we prefer them over OS pages as allocating large amounts of data
and then managing it at OS-page-granularity is a cost we aren't willing to pay,
although the OS I/O subsystem is normally still used for disk-writes.

Blocks themselves are structured such that the rows grow up towards the row
directory, table directory, and other metadata. Slotted pages make it so that
tuples can be inserted anywhere so long as the row directory is updated. 

We leverage `PCTUSED` and `PCTFREE` to avoid moving data around between pages
as that is an exceptionally expensive operation - they are used to determine 
state transitions in a simple state machine:

```python
STATE
def transition(state, capacity):
    if state == INSERT and capacity >= PCTFREE:
        state = UPDATE_ONLY

    if state == UPDATE_ONLY and capacity < PCTUSED:
        state = INSERT
```

Slotted rows mean that we can insert wherever we want into a block, and this
can lead to fragmentation. We mostly want to defer compaction for as long as
possible because it is expensive - for example doing it when there is 
enough space for an insertion or update but it is non-contiguous, forcing us
to move things around.

Free space is allocated at the segment level to avoid the complexity of doing
it at the extent level. We maintain one or more free lists - using more means
less contention.

Writing to disk is an art - various optimizations exist for updating tuples in
a way that is friendly for recovery and other concurrent users.

- **Shadow paging:** make copies of a page, and perform updates there. Update
directory with the new page after comitting. Aborting is free, but this 
mechanism destroys locality. We want to use this when copies are low-overhead.
- **Delta files:** Copy data into a special temp file, either make the 
modifications there or do them in-place, optimizing for aborts and commits
respectively.

## Buffer Cache

- Data got to be in-memory to be modified - but not everything can fit there!
- We want to keep important things in memory as it is a scarce resource 
relative to disk.
- Very complex system, lots of tunable params.
- Lock: mechanism to avoid conflicting updates to data by transaction
- Latch: mechanism to avoid conflicting updates to system data structures
- We don't want our latches to be too-fine grained. We can't be devoting too
much space to engine structures --> common trade-off in DBMS
- Latch contention can be a serious performance bottleneck on hot blocks
- We can reduce contention by reducing the amount of data stored on a block,
using more buffer pools, using finer grained latches, avoiding many concurrent
queries on the same data, ...
- We like having more hash buckets because then there are less linked-list 
entries to traverse. Also less contention! But this is, of course, a trade-off.
- Lots of metadata in block header, including buffer replacement information.
- Blocks can be pinned so that they aren't evicted, and have a usage count
for example (as well as clean / dirty flag)
- Various types of blocks in these linked lists _(version blocks, undo/redo
blocks for recover, dirty blocks, pinned blocks)_
- version blocks are a form of shadow paging 
- LRU: common strategy in OS, but doesn't relaly work in DBMS. E.g. table scan
flooding - we get a bunch of things into the cache that we aren't likely to 
re-use. Similary with an index range scan _(index pages in cache, which we 
might not need again anytime soon)_
- We can optimize by avoiding caching large pages, or putting things that are
rarely accessed at the bottom of the list instead of top.
- Oracle has a keep buffer pool reserved for important pages, and conversely a
recycle buffer pool. 
- We could also keep usage stats to know what should be cached and what 
shouldn't be - let the system automatically decide.
- Cache pollution is important to avoid - interacts with optimizations, 
sometimes caused by them
- Read-ahead uses plan semantics to find what's needed next. Can however cause
pollution!
- Clean pages quicker to evict than dirty page!
- Touch count: algo used by Oracle. Insert in middle, let hot pages float to 
top and cold pages sink to bottom. Have a cooldown to avoid ephemeral hot pages
polluting cache _(in the case that we only use them for a short period of 
time)_
- Clock sweep and second chance: more efficient than maintaining LRU list. 
Increment some counters up to a tunable max _(second chance has this to 1)_
- 2Q: FIFO list and LRU list. Evict from FIFO list

### Second pass - summary

This section goes over the various mechanisms in place for keeping our cache
as clean as possible, as our main memory is a scarce resource. The buffer cache
is a key component in any DBMS.

Firstly, the course slides cover latching in the buffer manager, noting that
DBMS distinguish between a lock _(avoid conflicting updates to data)_ and a 
latch _(avoid conflicting updates to system data structures)_. 

Determining the right latch granuarlity is a great example of a trade-off in 
DBMS, as high contention causes performance issues and more latches requires 
more resources allocated to DBMS infrastructure. A good example of this is
**hot blocks**, which are a source of contention _(think zipfian page access 
pattern)_. We can address this in various ways, each with their own respective
trade-offs

- Reduce the amount of data in a block _(in Oracle, configure `PCTUSED` and
`PCTFREE`)_ for example
- Configure the engine with more latches and less buckets per latch
- Multiple buffer pools
- Tune queries to minimize the number of blocks that can access, for example
avoiding table scans
- Avoid many concurrent queries over the same data
- Avoid concurrent transactions and queries against the same data

In general, the number of hash buckets maintained by the buffer manager should
be maximized, as a linked list of pages sits on the other side of it and we
want to avoid long traversals.

The block headers themselves, stored inside the aforementioned linked lists,
contain a bunch of metadata that differs between implementations. Importantly,
we normally keep some _status_ metadata that can be leveraged by cache 
replacement policies.

We note that we can also have version blocks _(and others)_ inside the linked
lists. Version blocks function similarly to shadow paging, i.e. we keep an 
older version of the data for undoing, or simply reading through the history.
Using version blocks can cause the linked list to grow rapidly, leading to
performance issues.

The buffer manager, of course, needs to choose a replacement strategy for its
cached pages. the course covers

- **LRU:** Not prefered at all. Very sensitive to pollution through events like
table scan flooding. There is no use of DBMS information in deciding what to
evict.
- **Modified LRU:** with slight modification, for example putting blocks that
are rarely accessed at the bottom of the LRU list so that they are evicted
quickly, or simply not caching large tables during scans or other.
- **Touch count:** insert in middle. Let the hot pages float up, and the cold
pages sink down. We have a cooldown after insertion to avoid pages that are
only accessed frequently for a short period of time.
- **Clock sweep:** a _hand_ goes around in a circular motion and decrements 
every page's counter and evicting when it reaches 0. The maximum is 
configurable, and a specific case of this is the **second chance** policy which
has this maximum tuned to 1. Blocks that are regularly accessed have a higher
chance of staying in memory.
- **2Q:** we maintain a FIFO _(blocks that do not need to be kept)_ and an LRU
_(blocks that are accessed several times)_. A FIFO block that is accessed again
is moved to the LRU. The block at the bottom of the LRU is moved to the FIFO.
We evict from the FIFO first.

Lots of optimizations are made, and the buffer replacement policy will 
inevitably interact with them and handle them in different ways.

- Cache pollution
- Read-ahead / prefetching

To summarize, the buffer cache and its efficient maintainance is vital for
achieving high performance in a DBMS, and we should not understate the overhead
related to maintaining the system data structures at its core.

## Paper 01: DBMIN

### Thousand foot view

- Key contribution: new algorithm for maintaining buffer pool
- Key contribution: query locality set model **QLSM** which is advantageous
over hot set model in that it separates behavior from any particular buffer
management algorithm.
- Key contribution: testing everything rigorously in a multi-user environment
- Lots of alternative algorithms proposed prior to this: _domain separation
(DS), Group LRU (GLRU), Working set (WS), "new" algorithm, Hot Set (HOT)_
- **HOT**: performance suffers when memory isn't large enough to hold the 
entire hot set due to lots of page faults. We call this point a hot point
- HOT doesn't reflect the inherent behavior of some access patterns, but rather
their behavior under LRU!
- HOT overallocates memory

### QLSM Reference types

We will break this down as it is one of the key points of the paper.

> Relational database systems support a limited set of operations and the
> pattern of page references exhibited by these operations are very regular and
> predictable.

#### Sequential

- **Straight sequential (SS):** think table scan. In most cases we access a 
page once and then never again
- **Clustered sequential (CS):** local re-scans occur _(small backup)_. This 
can happen during a merge join where the records with the same key value are
repeatedly scanned.
- **Looping sequential (LS):** sequential reference to a file repeated multiple 
times. Keep everything in the buffer if possible, otherwise use MRU.

#### Random

- **Independent random (IR):** Consists of a series of independent random 
accesses _(think index scan through a non-clustered index)_.
- **Clustered random (CR):** when the random accesses happen over some 
constrained subset of pages _(join containing an inner-relation with a 
non-clustered index and an outer relation is clustered with non-unique keys)_.
We want to be keeping each page containing a record in the cluster in memory.

#### Hierarchical

These are caused by sequences of page accesses that form a tree traversal from
root down to leaves of an index.

- **Straight hierarchical (SH):** we just traverse the index once to retrieve
a single tuple
- **Hierarchical with straight sequential (H/SS):** tree traversal followed by
sequential scan on a clustered index
- **Hierarchical with clustered sequential (H/CS):** tree traversal followed by
sequential scan on a non-clustered index
- **Looping hierarchical (LH):** _(when the inner relation of a join has an 
index, we end up accessing the index a bunch of times)_.

#### Fun note

Let $P_i$ denote the probability of accessing the $i^{th}$ level of a tree from
the root (which itself is at level $0$), and let $f$ be the fanout. Then

$$
P_i \propto f^{-i}
$$
