6.824 2016 Lecture 10: Distributed Optimistic Cocurrency Control

Today's Paper: Efficient Optimistic Concurrency Control using Loosely
Synchronized Clocks, by Adya, Gruber, Liskov and Maheshwari.

Why this paper?
  programmers like transactions, but hard to implement efficiently
  Thor gets speed from local client caches and no locking messages
    via optimistic concurrency control (OCC)

Thor overview
  [clients, client caches, servers A-M N-Z]
  data sharded over servers
  code runs in clients, not servers
  clients fetch/put objects (DB records) from servers
  clients cache objects locally for fast access
    on hit, client doesn't talk to server -- does not ask server to lock!
    on client cache miss, fetch from server

Client caching makes transactions tricky
  how to cope with reads of stale cached data?
  how to get concurrency control (since clients don't lock)?

A reminder:
  "serializable" = results as if xactions one at a time in some order
  "conflict" = two concurrent xactions used record, at least one wrote it

Thor uses optimistic concurrency control (OCC)
  an idea from the early 1980s
  just read and write the local copy
    don't worry about other transactions until commit
  when transaction wants to commit:
    send read/write info to server for "validation"
    validation decides if OK to commit -- if it can find equiv serial order
    if yes, update server's data, send cache invalidates to clients
    if no, abort, discard writes
  "optimistic" b/c executes w/o worrying about serializability
  "pessimistic" locking checks each access to force serializable execution

What should validation do?
  it looks at what committed/committing transactions read and wrote
  decides if there's a serial execution order that would have gotten
    the same results as the actual concurrent execution
  there are many OCC validation algorithms!
    i will outline a few, leading up to a simplified Thor
    FaRM uses yet another scheme

Validation scheme #1
  (this is a toy -- no-one validates using values)
  a single validation server
  clients tell validation server the read and write VALUES
    seen by each transaction that wants to commit
    "read set" and "write set"
  validation must decide:
    would the results be serializable if we let these
      transactions commit?
  scheme #1 shuffles the transactions, looking for a serial order
    in which each read sees the value written by the most
    recent write; if one exists, the execution was serializable.

Validation example 1:
  initially, x=0 y=0 z=0
  T1: Rx0 Wx1
  T2: Rz0 Wz9
  T3: Ry1 Rx1
  T4: Rx0 Wy1
  validation needs to decide if this execution (reads, writes)
    is equivalent to some serial order
  yes: one such order is T4, T1, T3, T2
    (really T2 can go anywhere)
    so validation can say "yes" to all four transactions
  note this scheme is far more permissive than Thor's
    e.g. it allows transactions to see uncommitted writes
    since all four ran concurently, none had committed at validation time,
      yet they evidently saw each others' writes
 
OCC is neat b/c transactions don't need to lock!
  so they can run quickly from client caches
  just one msg exchange w/ validator per transaction
    rather than one locking exchange per record used
  OCC excellent for T2 which didn't conflict with anything
    we got lucky for T1 T3 T4, which do conflict

Validation example 2 -- validation sometimes fails:
  initially, x=0 y=0
  T1: Rx0 Wx1
  T2: Rx0 Wy1
  T3: Ry0 Rx1
  how do the values constrain potential equivalent serial orders?
    T1 -> T3 (via x)
    T3 -> T2 (via y)
    T2 -> T1 (via x)
  there's a cycle, so not the same as any serial execution
  perhaps T3 read a stale y=0 from cache
    or T2 read a stale x=0 from cache
  in this case validation can abort one of them
    then others are OK to commit
  e.g. abort T2
    then T1, T3 is OK (but not T3, T1)

How should client handle abort?
  roll back the program (including writes to program's memory!)
  re-run from start of transaction
  hopefully won't be conflicts the second time
  OCC is best when conflicts are rare!

Do we need to validate read-only transactions?
  example:
    initially x=0 y=0
    T1: Wx1
    T2: Rx1 Wy2
    T3: Ry2 Rx0
  i.e. T3 read a stale x=0 from its cache, hadn't yet seen invalidate.
  no serial order gets the same results
  so we do need to validate r/o transactions
  other OCC schemes can avoid validating read-only transactions
    by keeping multiple versions -- but Thor and my schemes don't

Is OCC better than locking?
  yes, if few conflicts
    avoids lock msgs, clients don't have to wait for locks
  no, if many conflicts
    OCC aborts, must re-start, perhaps many times
    locking waits, doesn't need to re-execute
  example: simultaneous increment
    locking:
      T1: Rx0 Wx1
      T2: -------Rx1  Wx2
    OCC:
      T1: Rx0 Wx1
      T2: Rx0 Wx1
      fast but wrong; must abort one

We want *distributed* OCC validation
  split storage and validation load over servers
  each storage server validates just its part of the xaction
  each transaction managed by a transaction coordinator (TC)
    perhaps each client acts as its own TC
  TC asks each server to validate,
    then (if all yes) tells each server to commit
    this part is two-phase commit (but w/o locks)

Can we just distribute validation scheme #1?
  i.e. each server says "yes" if equivalent serial order for its objects
  imagine server S1 knows about x, server S2 knows about y
  example 2 again
    T1: Rx0 Wx1
    T2: Rx0 Wy1
    T3: Ry0 Rx1
  S1 validates just x information:
    T1: Rx0 Wx1
    T2: Rx0
    T3: Rx1
    T2 T1 T3 is equivalent, so S1's answer is "yes"
  S2 validates just y information:
    T2: Wy1
    T3: Ry0
    T3 T2 is equivalent, so S2's answer is "yes"
  but we know the real answer is "no"

So naive distributed scheme #1 does not work
  the validators must choose consistent orders!

Validation scheme #2
  Idea: client (or TC) chooses timestamp for committing xaction
    from loosely synchronized clocks, as in Thor
  validation checks that reads and writes are consistent with TS order
  solves distrib validation problem:
    timestamps tell the validators the order to check
    so "yes" votes will refer to the same order

Example 2 again, with timestamps:
  T1@100: Rx0 Wx1
  T2@110: Rx0 Wy1
  T3@105: Ry0 Rx1
  S1 validates just x information:
    T1@100: Rx0 Wx1
    T2@110: Rx0
    T3@105: Rx1
    timestamps say order must be T1, T3, T2
    validation fails! T2 should have seen x=1
  S2 validates just y information:
    T2@110: Wy1
    T3@105: Ry0
    timstamps say order must be T3, T2
    validates!
  S1 says no, S2 says yes, TC will abort

Timestamps ensure different validating servers are all checking
  the *same* order (TS order) during validation.

What have we given up by requiring timestamp order?
  example:
    T1@100: Rx0 Wx1
    T2@50: Rx1 Wx2
  T2 will abort, since TS says T2 comes first, so T1 should have seen x=2
    could have committed, since T1 then T2 works
  this will happen if client clocks are too far off
    if T1's client clock is ahead, or T2's behind
  so: requiring TS order can abort unnecessarily
    b/c validation no longer *searching* for an order that works
    instead merely *checking* that TS order consistent w/ reads, writes
    we've given up some optimism by requiring TS order
  maybe not a problem if clocks closely synched, as in Thor
  maybe not a problem if conflicts are rare

Problem with schemes so far:
  commit messages contained *values*, which can be big
  could instead use per-object version numbers to check whether
    later xaction read earlier xaction's write
  let's use writing xaction's TS as object's version number
    could use sequential numbers instead, as in FaRM

Validation scheme #3
  tag each DB record (and cached record) with TS of xaction that last wrote it.
  validation requests carry TS of each record read.
  check that each read saw the version # written by most recent writing xaction
    in timestamp order.
  commit sets written objects' versions equal to transactions TS.

Our example in scheme #3:
  all values start with timestamp 0
  T1@100: Rx@0 Wx
  T2@110: Rx@0 Wy
  T3@105: Ry@0 Rx@100
  note:
    read-set has versions of read objects, not values
    write-set contains new values, but not used by validation
  S1 validates just x information:
    orders the transactions by timestamp:
    T1@100: Rx@0   Wx
    T3@105: Rx@100
    T2@110: Rx@0
    the question: does each read see the most recent write?
      T3 is ok, but T2 is not, so S1 says "no"
  S2 validates just y information:
    again, sort by TS, check each read saw latest write:
    T3@105: Ry@0
    T2@110: Wy
    S2 says "yes"
  so scheme #3 aborts, correctly, reasoning only about version #s and TSs
  this is half the answer to the Thor Question:
    Q: what does Thor do if a transaction reads stale data?
    A: if Thor used version numbers, they would tell validation that
       T2 had read a stale copy of x. Section 3.3's "version check".

what have we give up by thinking about version #s rather than values?
  maybe version numbers are different but values are the same
  e.g.
    T1@100: Wx1
    T2@110: Wx2
    T3@120: Wx1
    T4@130: Rx1@100
  versions say we should abort T4 b/c read a stale version
    should have read T3's write
    so scheme #3 will abort
  but T4 read the correct value x=1
    so abort wasn't necessary
  this is OK in practice; many OCC systems use version #s

Problem: per-record version # might use too much storage space
  Thor wants to avoid space overhead 
  maybe important, maybe not
  
Validation scheme #4
  Thor's invalidation scheme: no timestamps on records
  how can validation detect that a transaction read stale data?
  it read stale data b/c earlier xaction's invalidation hadn't yet arrived!
  Thor server tracks invalidation msgs that each client might not have seen
    "invalid set" -- one per client
    delete invalid set entry when client ACKs invalidation msg
    server aborts committing xaction if it read record in client's invalid set
    client aborts running xaction if it read record mentioned in invalidation msg

Example use of invalid set
  [timeline]
  Client C2:
    T2 ... Rx ... client/TC about to send out prepare msgs
  Server:
    T1 wrote x, validated, committed
    x in C2's invalid set on S
    server has sent invalidation message to C2
    server is waiting for ack from C2

Three cases:
  inval arrives at C2 before C2 sends prepare messages
    C2 aborts T2
  inval arrives after prepare sent, while C2 waiting for yes/no
    C2 aborts T2
  inval dropped/delayed (so C2 doesn't know to abort T2)
    S can't have heard ack from C2
    C2 still in x's inval set on S
    S will say "no" to T2's prepare
  so: Thor's validation detects stale reads w/o timestamp on each record
  this is the full answer to The Question

Performance

Look at Figure 5
  AOCC is Thor
  comparing to ACBL: client talks to srvr to get write-locks,
   and to commit non-r/o xactions, but can cache read locks along with data
  why does Thor (AOCC) have higher throughput?
    fewer msgs: commit only, no lock msgs
  why does Thor throughput go up for a while w/ more clients?
    apparently a single client can't keep all resources busy
    maybe due to network RTT?
    maybe due to client processing time? or think time?
    more clients -> more parallel xactions -> more completed
  why does Thor throughput level off?
    maybe 15 clients is enough to saturate server disk or CPU
    abt 100 xactions/second, about right for writing disk
  why does Thor throughput *drop* with many clients?
    more clients means more concurrent xactions at any given time
    more concurrency means more chance of conflict
    for OCC, more conflict means more aborts, so more wasted CPU
  
Conclusions
  caching reduces client/server data fetches, so faster
  distributed OCC avoids client/server lock traffic, again for speed
  loose time sync helps servers agree on equivalent order for validation
  distributed OCC still a hot area 20 years later!