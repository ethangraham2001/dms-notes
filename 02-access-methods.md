# Access Methods

## Introduction and motivation

This level lies on top of the pages-in-memory. Here, the goal is to discuss how
we go about accessing the data that we want.

- Main question: how is data represented and organized in-memory?
- Block structure and indexes
- Different workloads: OLTP, OLAP
- OLTP: short transactions with updates, point queries
- OLAP: dominated by complex queries
- We will discuss trade-offs. Column store? Row store? Keep unused space for 
    updates and movements?

## Pages and blocks

### Thousand-foot

- **Recall:** blocks, extents, segments...
- We trend towards larger blocks as it incurs less overhead
- Finding the page you need: managed through lists _(stored in segment header)_
    note that this can be a bottleneck! Keep the lists short and sorted.
- Finding tuples within a page - slotted pages. Page maintains a list of slots.
    Elegant handling of variable-length tuples.
- **Recall:** `PCTFREE`, `PCTUSED` state machine. We don't want too many 
    pointers to other blocks. Keep data nice and local. `PCTFREE` to avoid 
    fragmentation / indirection, `PCTUSED` to avoid constantly moving a block 
    to the free list and increasing maintainance cost.
- Every tuple contains: header _(flags)_, data for each non-null attribute, or 
    a pointer. We don't store schema information in the tuple.
- Sometimes we want to store offsets, as this allows for linear access to the
    attributes.
- BLOBs: we would rather store a pointer in this case. These objects can be
    much larger than a single block, i.e. in the case of a page.
- NSM/row store: pretty intuitive, and great for accessing a whole tuple. This
is targetted at OLTP where we interact normally with a single **whole** tuple 
    at a time.
- Row store cumbersome for complex queries.
- Column store is great for vectorized computing _(e.g. SIMD)_ as it optimizes
    for this type of storage... Addresses the memory wall. Better cache 
    utilization
- Column store suffers when most of the attributes are needed anyways, or when
    the tuples / parts of them need to be reconstructed.
- PAX is a hybrid between NSM and column-store approaches.
- Compression: trade-off CPU cycles for better bandwidth utilization. e.g.
    dictionary compression, RLE, delta encoding, bitmaps
- Dictionary compression: e.g. map country-codes to integers
- Delta encoding: values have locality, so we can reconstruct from an offset
    easily
- RLE: values repeated a lot, e.g. long strings with repeating chars
- Bitmaps: make a bitmap for every value that an attribute might take. Can be
    used as an index. Can further compress using RLE, however increases query
    complexity.

### Second Pass: Summary

In this chapter we discuss how records can be laid out on a page. We start by
having a look at slotted pages, which is a pointer-based approach so that we
can freely move records around within a tuple and only have to update a pointer
in a header.

We then recall tunable parameters liks `PCTUSED` and `PCTFREE`, which was 
discussed in a previous lecture. We motivate these parameters more here,
observing that `PCTFREE` serves to avoid fragmentation when pages don't have
enough free space to for an update, and observing that `PCTUSED` helps avoid
having to constantly move a block from the used list to the free list and vice 
versa. These two values function sort of like a state transition in a 
state machine that tells us whether we are allowed to update or insert or both.

Next, we discuss the actual record layout in a page, and some optimizations
that can be done such as storing offsets instead of lengths to turn a linear
time lookup into a constant time one, and storing variable length fields at the
end of the record. We highlight how in a record, we might not actually want the
data to live in the tuple, e.g. for an image where we would rather have a 
BLOB URI than try and store large amounts of data in a page which is totally
unfeasible.

We then discuss different ways partitioning records across pages. For exmaple,
**Row/N-ary storage model** where we put a whole tuple in a single page, and
**column-store** where a single page will only hold attributes of one type for
different records _(requires reconstruction for full record)_ and PAX which
sits in the middle. They all optimize for different things. We make a strong
statement which is **Row store is for OLTP, column store is for OLAP**.

We note that modern systems like using vectorization heavily, which is very
well-suited for column-stores - everything is laid out just as it should be.
However column stores suffer when we need most attributes of a tuple, most of
the time. PAX is an alternative representation that addresses this, sitting
somewhere between row-store and column-store, using minipages that are cache
friendly _(column-store fashion)_ while keeping all data within the same page
_(row-store fashion)_.

We finally look at compression broadly, stating that it trades off CPU cycles
for bandwidth and storage capacity, and mention some common compression 
approaches that are used in DBMS.

## Indexing

### Thousand foot

- We primarly look at Hashing and B+ trees.
- Indexes typically are not clustered - i.e. data stored in the extents is
typically not organized according to the index.
- We like clustered indexes.
- Hashing: trade-off storage vs. compute. B+ trees sacrifice space to speed up
the search. Hashing uses compute _(hash function)_ to find slot of where 
something is stored.
- Limitation of hashing: perfect hash functions exist, but you need a hash 
table as big as the cardinality of the attribute! Not great. 4-byte keys could
mean a 4GB table. Moreover, they only support point queries.
- Chaining: how we handle collisions.
- Open addressing, one solution for collision handling. We can also use cuckoo
hashing wherein we use several hash functions if we collide with a value using
the first.
- Growing pains: growing a hash table isn't fun. We don't want to have to build
a new hash table whenever we need to grow.
- Extensible hashing: handles overflow gracefully. We can logically double the
size of the hash table without double the amount of space being used. We can
grow the bucket directory independently of the data blocks.
- Linear hashing: split pointers and all that. Gradually increase he size of 
the table and redistribute the data.
- B-tree: has order `k` and each node except root has `[k/2, k]` child nodes.
Root has at least 2 children.
- Today, DBMS only use B+ trees, not B-trees.
- B+ tree is a B-Tree but the data is only in the leaves, and they are 
balanced.
- Typically, the lead nodes contain pointers, and the inner node values contain
separators - not actual key values.
- Typically, indexes only clustered for the primary key.
- B+ tree can use one or more attributes as the key to the index - composite
keys!
- Can build B+ trees on non-unique values, of course. However, this means
that we have to handle duplicates. _(repeat key for duplicated entries, or 
store the key once and point to a linked-list matching all entries)_
- Range scans are nice in B+ trees.
- Updates to the index are a pain in the ass
- **Concurrent access:** where do the locks go?? How many locks do I need to
hold?
- Lock coupling: lock page and its parent, however not enough if we have to go
further up...
- We can also use coupling, and then always sensure there is space for one more
entry. If not, just split the node _(we checked the parent, so all good)_.
This approach ensures that split never causes changes to be propagates to the
root.
- We can create a B+ tree by bulk inserting data: sort it, and then build tree
from the bottom up. This results in a clustered and compact tree. If we want to
update the tree, better leave some space in each block!
- Reverse indexing is cool - i.e. make `1234` and `1235` land on different 
pages by inserting them as `4321` and `5321`. I guess we can avoid some
contention, but this seems suboptimal to me...
- We can do some optimizing by factoring common prefixes, since values next to
each other are likely to be similar _(think in terms of strings here)_.
- Sometimes we want to ignore B+ tree rules and defer merging. Instead, we
periodically rebuild the tree.
- If nodes are large, tree is shallow, and this is good for slow storage 
devices _(sequential access on the nodes, we like this)_
- If nodes are small, then the tree ends up being deeper. This is nice for fast
storage.
- Trees treated as first-class citizens. Often, queries can be answered just
be looking at the index and not even looking at the data itself.
`SELECT COUNT(*) FROM table WHERE age > 5 AND age < 10`, where we don't even
need to go ahead and read the actual data.
- Query selectivity: important, as indexes work when there is high selectivity.
Table scans cheap is table is small, and could be faster than indexing!
- Bitmaps: enumerate all possible values that an attribute can take. Simple
predicate selections can be made very efficient with this. Sparse bitmaps
_(with lots of `0` values)_ compresses really nicely with RLE.
- Specialized indexes are a thing, e.g. Tries, R trees, grid files, ... list
goes right on.

### Second pass: summary

This section broadly covers indexing, which is a mechanism for speeding up
various forms of DBMS queries - tracing off space for speed. In particular, we
look at hash indexes and B+ Trees.

We define clustering, noting that most indexes are unclustered.

We then look at various designs for hash tables, noting chaining, open 
addressing, including designs that gracefully handle extensibility, such as
extensible hashing and linear hashing.

As for B+ Trees, in particular we look at composite indexes, noting pretty much
the same thing as in Big Data - we can use a prefix of the index keys on an
index, but not just any arbitrary value _(unless it is the first)_. We note 
that we can build a B+ Tree on a non-unique attribute if we want to, howver
we have to specify how to handle this at the tree-level _(repeat key? point to
a list of matching entries)_. Leaves have sibling pointers which speeds up
scans once we have traversed the tree into the root.

Concurrent access in indexes is a trade off - we may have many concurrent 
threads making updates and reading, and we must decide at what granularity we
want to lock.

We also discuss some optimizations that can be done for B+ trees, such as
bulk inserting into a tree _(like batching)_, or using things like reverse
indexes which prevent values like sequence numbers that monotonically increase
from causing high contention when lots of inserts are made at once - this, of
course, trades off being able to perform range queries, but these aren't really
used for sequence numbers anyhow. Other optimizations include shorter 
separators in B+ tree inner nodes _(in the case of strings, for example, we 
don't want to perform long `strcmp`)_ or just factoring common prefixes _(e.g.
`bdsb, bdsm, bdsr -> bds(b, m, r)`)_ for the same reason. We can also choose
to ignore some of the B+ tree rules, defering merges or periodically rebuilding
tree, to optimize to low-latency operations most of the time.

We explained last chapter that _segments_ are used to refer to DBMS entities
like tables and indexes. Indexes are treated as first-class citizens in DBMS,
like tables. Often we can simply answer queries by just looking at the index
instead of performing any actual data lookup.

If query selectivity is high enough, we don't necessarily want to be looking
up an index at all, and just perform a full able scan.

Sometimes we want to materialize aggregates if we know that they will be used
super frequently. This is functionally just a very small index that we use to
quickly lookup precomuted aggregated.

### Thousand foot

- The ideas discussed in previous chapters are deeply interconnected. We 
wouldn't want a B+ tree in a column oriented DB, and column-oriented DBs use
compression very heavily.
- Access to disaggregated storage is very expensive, and blocks are larger. 
Large tables are normally partitioned!
- **Normalized tables:** split distinct concepts into their own tables. 
However, this leads to redundancy and we do lots of joins!
- **Clustered tabled:** cluster tables into the same segment for locality, and
index them on the common attributes/keys. Oracle does this, and it makes sense
when the tables are processed together most of the time. Reduces I/O and saves
space. Functions like a materialized `JOIN`. Note that updates can become
expensive.
- **Log structured file:** don't store tuples and update them as needed, keep
an update of how the tuples were modified in a log. We only record the 
information that is needed _(deletion invalidates, insertion records whole 
tuple, and updates only record the updated section)_. Append-only.
- Sequential file updates are much faster than random access even on SSDs, and
this is the key idea behind a log-structured DB. Minimize cost of making data
persistent, but this is very expensive for OLTP when there are many 
transactions.
- We can optimize by periodically compacting the log, i.e. periodically apply
all updates. Increasingly being used in cloud DB.
- **Snowflake:** data warehouse for analytical queries, and cloud-native.
Separate compute-nodes from storage.
- Snowflake uses common design traits: horizontal partitioning _(by row)_,
columnar format _(best for analytics, cache-friendly, can be vectorized)_,
and storage-level processing allowing us to only read relevant parts of a file.
- **Pruning:** snowflake doesn't use indexes as they require a lot of space
and random access _(the latter of which is catastrophic for slow storage like
S3)_. Instead, uses metadata for filtering micro-partitions - e.g. header
contains the min and max values of an attribute, making
`SELECT * FROM table WHERE age > 45` nice and easy.
- Writing to disk is tricky because S3 is immutable _(no in-place update)_,
so snowflake leverages this to snapshot data _(like shadow paging)_. Can thus
time travel, and provides fault-tolerance.
- **Database cracking:** create no index - instead build the index 
incrementally while the data is being processed. Initial queries are expensive,
then subsequent cost is amortized as the hard work has already been done.
