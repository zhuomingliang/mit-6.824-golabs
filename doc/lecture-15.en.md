6.824 2016 Lecture 16: 

Wormhole: Reliable Pub-Sub to support Geo-replicated Internet Services, Sharma
et al, 2015.

why are we reading this paper?
  pub-sub common building block in distributed systems
    YMB, FAB, Kafka
  case study: Facebook's Wormhole
    motivated by memcache

how do web sites scale up with growing load?
  a typical story of evolution over time:
  1. one machine, web server, application, DB
     DB stores on disk, crash recovery, transactions, SQL
     application queries DB, formats, HTML, &c
     but the load grows, your PHP application takes too much CPU time
  2. many web FEs, one shared DB
     an easy change, since web server + app already separate from storage
     FEs are stateless, all sharing (and concurrency control) via DB
     but the load grows; add more FEs; soon single DB server is bottleneck
  3. many web FEs, data sharded over cluster of DBs
     partition data by key over the DBs
       app looks at key (e.g. user), chooses the right DB
     good DB parallelism if no data is super-popular
     painful -- cross-shard transactions and queries probably don't work
       hard to partition too finely
     but DBs are slow, even for reads, why not cache read requests?
  4. many web FEs, many caches for reads, many DBs for writes
     cost-effective b/c read-heavy and memcached 10x faster than a DB
       memcached just an in-memory hash table, very simple
     complex b/c DB and memcacheds can get out of sync
     (next bottleneck will be DB writes -- hard to solve)

the big facebook infrastructure picture
  lots of users, friend lists, status, posts, likes, photos
    fresh/consistent data apparently not critical
    because humans are tolerant?
  high load: billions of operations per second
    that's 10,000x the throughput of one DB server
  multiple data centers (at least west and east coast)
  each data center -- "region":
    "real" data sharded over MySQL DBs
    memcached layer (mc)
    web servers (clients of memcached)
  each data center's DBs contain full replica
  west coast is master, others are slaves via MySQL async log replication

what does FB store in mc?
  maybe userID -> name; userID -> friend list; postID -> text; URL -> likes
  basically copies of data from DB

how do FB apps use mc?
  read:
    v = get(k) (computes hash(k) to choose mc server)
    if v is nil {
      v = fetch from local DB
      set(k, v)
    }
  write:
    v = new value
    send k,v to master DB  # maybe in remote region

how to arrange that DB in different regions get updated?
  master DB receives all writes (similar to PNUTS)
  adds entry to transaction log
  replicates transaction log to slaves

how to arrange that mc in different regions learn about writes
  need to invalidate/update mc entry after write
    eval section suggests it is important to avoid stale data
  option 1: mc in remote region polls its local DB
     increases read load on DB
     what is the poll interval?
  option 2: wormhole pub/sub

pub/sub
  a common building block in distributed systems
  facebook use case
    subscriber is mc
    publisher is DB
  subscribers link we a library
    update configuration file to express interest in updates
    stored in zookeeper
  publishers read a configuration file to find subscribers
    establishes a flow with each subscriber
    send wormhole updates on each flow asynchronously
      set of key-value pairs
  filters
    subscribers tell publishers about a filter
    filter is a query over keys in wormhole update
    publishers send only updates that pass filter

delivery semantics
  all updates on a flow are delivered in order
  publisher maintains per subscriber a "data marker"
    sequence number of an update in the transaction log
    records what a subscriber has received
  publishers ask subscriber periodically for what it has received
    i.e., marker is a lower bound what subscriber has received
  updates are delivered at least once
    publisher persists marker
    if publisher fails start sending from last marker
    => subscribers may receive update twice
  Q: how do subscribers deal with an update delivered several times
    A: no problem for caches
    A: application can do duplicate filtering
  Q: can an update be never delivered?
    A: yes, because transaction log may have been truncated
       data in log is present for 1-2 days
       
Q: why does subscriber not keep track of marker?
   A: FB wants subscribers to be stateless

Q: why are markers periodically acked by subscribers
   A: Expensive to ack each update
   A: Uses TCP for delivery
     They don't have to worry about packet loss

Where to store the marker?
  SCRD: publisher stores in local persistent storage
    if storage is unavailable, cannot fail over to new publisher
       caches will be stale
    if storage fails, lose marker
    opportunity:  storage/log is often replicated
  MCRD: publishers stores marker in Zookeeper
    if publisher fails, another publisher can take over
    read last marker from zookeeper
    implementation challenge: format of marker
      replicas of same log have different binary format
      solution: "logical" positions
  Q: isn't it expensive to update marker in Zookeeper?
    A: yes, but only done periodically

Implementation challenge: many different DBs
  don't want to modify any of them to support flows
  idea: publishers read transaction log of DB
    read library to read different log formats
    convert updates in standard format (Wormhole update)
    one key is a shard identifier

Optimization 1: caravan
  design 1: one reader per flow
    puts too much load on DB
    in steady state, all readers read same updates
  design 2: one reader for all flows
    bad performance on recovery
    on recovery each flow may have to read from different point in log
  solution: one reader per cluster of flows ("caravan")
    in practice, number of caravans is small (~1)
    that one caravan is called the "lead caravan"
    
Optimization 2: load balancing flows
  a single application has several subscribers
    application data is sharded too
    N DB shards
    M application machines
    -> an MC machine may have more than 1 subscriber
       e.g., when it stores 2 DB shards
    Q: on creation of a new flow which app machine is the subscriber? 
  two plans:
    - weighted random selection
    - subscribers use zookeeper

Optimization 3: one TCP connection
  multiplexes several flows for same subscriber
  subscriber may have several shards
  one flow for each shard

Deployment
  in use at FB
  readers for several DBs (MySQL, HDFS, ...)
  35 Gbyte/s of updates per day
  # caravans = 1.06    

Performance
  Publisher bottleneck: 350 Mbyte/s
  Subscriber bottleneck: 600,000 updates/s
  Enough for production workloads

References
  Kafka (http://research.microsoft.com/en-us/um/people/srikanth/netdb11/netdb11papers/netdb11-final12.pdf)
