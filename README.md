# ReadDB
I'm interested in database systems and I plan to learn this field broadly by reading papers, so I make a repo as a notebook.  
Since I focus on things I'm interested to learn, the notes may omit many details of the papers.

## Abbreviation
Transaction - Tx  
Concurrency Control - Cc  
Repeatable Read - RR  
Snapshot-Isolation - SI  
Highly Available/High Availability - HA

## List
- [x] TiDB
- [x] Bigtable
- [x] Pavlo, A., Curino, C. and Zdonik, S., 2012, May. Skew-aware automatic database partitioning in shared-nothing, parallel OLTP systems. In Proceedings of the 2012 ACM SIGMOD International Conference on Management of Data (pp. 61-72).
- [x] Zhou, X., Yu, X., Graefe, G. and Stonebraker, M., 2023. Two is Better Than One: The Case for 2-Tree for Skewed Data Sets. memory, 11, p.13.
- [ ] Dynamo
- [ ] O’Neil, P., Cheng, E., Gawlick, D. and O’Neil, E., 1996. The log-structured merge-tree (LSM-tree). Acta Informatica, 33, pp.351-385.
- [x] Franklin, M.J., 1997. Concurrency Control and Recovery.
- [ ] Vuppalapati, M., Miron, J., Agarwal, R., Truong, D., Motivala, A. and Cruanes, T., 2020. Building an elastic query engine on disaggregated storage. In 17th USENIX Symposium on Networked Systems Design and Implementation (NSDI 20) (pp. 449-462).
- [x] Li, G., Dong, H. and Zhang, C., 2022. Cloud databases: New techniques, challenges, and opportunities. Proceedings of the VLDB Endowment, 15(12), pp.3758-3761.
- [ ] Li, G., Zhou, X. and Cao, L., 2021, June. AI meets database: AI4DB and DB4AI. In Proceedings of the 2021 International Conference on Management of Data (pp. 2859-2866).
- [ ] Perron, M., Castro Fernandez, R., DeWitt, D. and Madden, S., 2020, June. Starling: A scalable query engine on cloud functions. In Proceedings of the 2020 ACM SIGMOD International Conference on Management of Data (pp. 131-141).
- [ ] Verbitski, A., Gupta, A., Saha, D., Brahmadesam, M., Gupta, K., Mittal, R., Krishnamurthy, S., Maurice, S., Kharatishvili, T. and Bao, X., 2017, May. Amazon aurora: Design considerations for high throughput cloud-native relational databases. In Proceedings of the 2017 ACM International Conference on Management of Data (pp. 1041-1052).
- [x] Stonebraker, M. and Hellerstein, J., 2005. What goes around comes around. Readings in database systems, 4, p.1.


## TiDB

A Distributed SQL(aka. NewSQL) system supporting HTAP workloads.

The need for HTAP: Building two systems separately for OLTP and OLAP is just more complicated than supporting both in one system. In addition, the freshness of data read by OLAP engine is important.

Two important concepts in HTAP: _Freshness_ (how recent the data read by analytical queries is) and _isolation_ (avoid performance interference between OLTP and OLAP workloads).

Most important innovation: by introducing a _learner_ role in Raft algorithm which replicates logs from leader to generate columnar data for OLAP. The learner doesn't count into the quorum of Raft requests, nor does it participate in the election process.  Learner asynchronously replicates data from leader.

Row store is called TiKV, column store is called TiFlash. TiKV is based on RocksDB. It stores ordered maps(probably we can just say SSTable) and does range paritions. Each partion is managed by a Raft group, each TiKV server is a node(follower or leader) in the Raft group. So if there are 3 TiKV servers and data is split into 3 partitions, logically there will be 9 paritions in total, since each partition is replicated on all 3 servers. Writes go to leader, which propagates log entries to followers and wait for quorum success. It does some optimizations on Raft execution:  
1. Assume log append success and keep sending new entries before receiving success, if fail just resend.
2. Leader parallelizes appending entries to logs(disk I/O) and sending logs to followers(network latency).
3. Applying changes after append success is also handles by another thread.
4. Follower can also respond to reads if its commit idx >= leader's commit.

Column store is called TiFlash. The logs it learns from the leader are in the format of DML transaction operations(like WAL) including insert, update and delete. An optimization is to compress logs based on TX status and overwritten tuples. TiFlash uses an interesting storage model called Delta Tree. It separates normal tuples and incoming deltas into two spaces, and periodically merges deltas with the tuples. Since incoming deltas are unsorted, it builds a B+ Tree index for the deltas to efficiently merge them. It argues that the read performance of Delta Tree is twice faster then LSM Tree used in TiKV. (What I feel is Delta Tree combines the properties of the two classic storage models - B+ and LSM)

Tx processing: Cc protocol is based on MVCC, but it also provides pessimistic control. TiDB prodives SI or RR. Distributed tx in coordinated by 2PC. The difference on implementaion between optimisitic and pessimistic here is when to acquire locks. For the former, locks are acquired after DMLs complete, when data is writing to TiKVs; for the latter locks are acquired as DMLs are executed in local space.

OLAP optmizations: cropping unneeded columns, eliminating projection, pushing down predicates, deriving predicates, constant folding, eliminating “group by” or outer joins, and unnesting subqueries.


## Bigtable

Target: Scalable, HA, Widely applicable, High performance  

Wide-column, update to a single row is atomic even for multiple columns.  

Sorted String Table with range parition, so client can design row keys such that frequently used-together keys are close to each other. A row range is called tablet.  

Architecture is one master server + many tablet servers, master manages distribution of tablets. Bigtable uses a tree-like structure, stores location of tablet metadata at the root, which stores location of tablets, which store the actual data. Client usually doesn't contact master node, instead uses a DNS-like fashion to lookup and cache tablet locations.  

Writes firstly go to logs on GFS, then go to the memtable, then go to SSTable. The metadata mentioned above contains sstable + log of a tablet, so using the metadata can recover a tablet.
Reads go to the merge view of memtable and sstable.  

Details about SSTable are omitted since I've read about it elsewhere. Immutability makes CC very easy.  

Highlights from benchmarking: Writes are faster than random reads since commits are grouped and the append-only nature. Sequential reads are much faster than random reads due to block-level caching. Load-balancing is not good, which hurts scalability.


## Pavlo12

Two problems in distributed SQL systems: number of distributed transactions and skew. The goal is to generate good database design that minimizes both through automatic partitioning.  
Design options include:  
1. Horizontal partitioning: fragmenting a table by some attributes.
2. Table replication: replicating some read-only/heavy tables on all nodes to save coordination, since it's always available locally from any node's point of view.
3. Secondary index: building index for more columns to reduce accesses of more kinds of queries.
4. Stored procedure routing: send query to the node that has most of the data needed by the query.

Use a large-neighborhood search based on cost model which calculates:

1. CoordinationCost, which is an equation that multiplies number of distributed transaction and number of parition accessed
2. Skew factor, (in brief)which computes the access rate(NumUsedByTx/SumNumUsedByTx) for each partition, then adds up the ratio between this rate and ideal rate(1/NumPartition)

## Li22

Cloud-native OLTP: Disaggregate compute and storage to scale them independently. Log is the database. Log can be separate from storage so that log persists writes and storage serves reads. A shared buffer pool layer can be added to further improve read performance and reduce duplicated values read by compute nodes.

Cloud-native OLAP: Similar to the disaggregation of OLTP. Compute nodes might cache hot data in SSD cache, shared memory can be used to accelerate distributed joins.

Two kinds of serverless: function as a service, and elastic query engine.

Challenges: Support multiple write nodes, fine-grained serverless provisioning, cloud-native HTAP(but seems there exists solutions, such as singlestore and polarDB)

## Franklin97

For me this is more like a review of the database course I took, so I don't want to replicate the class notes here. This paper is great - it explains CC and recovery very well, clears some of my confusions.


## Zhou23

Two tree architecture: one tree for hot data, one tree for cold data. This improves memory utilization because usually one block in a B+Tree has many cold records and few hot ones. This paper proposes a record migration protocol which essentially aggregates hot records into blocks and store in the hot tree and prioritizing accessing hot tree, so blocks with full hot data are likely to stay in memory, which improves mem utilization.

Migration is bi-directional: downward(from hot tree to cold tree) and upward(cold tree to hot tree). Downward is pretty much like cache eviction. To reduce metadata overhead brought by policy like LRU this paper uses CLOCK replacement. Upward migration is pretty simple, it's a naive approach(move data upward once it's accessed) combined with a sampling rate D, which probabilistically determine if to move(random(0,1) <= D). The reason to do a sampling is to prevent cold data from replacing hot data, for example sequential scan. It's very simple, comes with the benefits of nearly zero metadata overhead.

vs LSM tree: It only migrates downwards, not upwards, so this paper supports upward migration for RocksDB to do benchmarking.  
vs Naive row cache: It takes space, not good for range query.  
vs Anti-Caching: It consumes much more memory to maintain metadata.  

Durability and recovery is easy, just use existing approach.  

Highlights from benchmarking: Even when hot dataset is small enough to fit in memory under traditional B+Tree, two-tree is still faster because hot tree's index node is small. For update workload, naive LSM Tree can be as good as LSM Tree+upward migration because update workload naturally clusters hot records in top level.
For range scan workload, there are not much improvements(I think mostly due to the fact that hot tree should be small so still needs to read from bottom).  

Future directions: CC, Cloud-native(exploiting shared buffer pool), non-tree based data structures.


## Stonebraker05

This is a longer paper that covers many data models appeared in the history. Some highlights:

1. KISS. If complexity cannot bring _siginifcant benefits_, it's likely to lose to simple solutions. In earlier IMS and CODASYL(which is like what we call "graph database" now), the idea to use tree or graph based data structure to represent "relations" turn out to be too complex, because the data structures themselves have natural constraints(e.g. parental relations in tree, multi-dimensionality of graph). These constraints 1. make query harder 2. decrease independence 3. cause duplication 4. make database design harder.
2. Independence. The property of keeping applications to run even if schema changes is important. I still didn't fully understand why relational model has better logical data indepence than CODASYL/graph. Changing schema of a table can also make some SQL queries in the application code not runnable. I guess it's because changing a table schema has smaller affects than changing a type in the graph network, sicne graph queries are likely to involve many more type names/attribute names than relational queries.
3. Flexibiltiy/generalization. The only successful innovation after relational is Object-Relational database(Postgres), featured by its UDT/UDF system. The idea of supporting new types and specialized access methods for some certain use cases(in order to gain performance) is not good, because there will always be new types and new demands in the future. If we implement this in DBMS, the work is never gonna end. So rather, just let application developers do it themselves.
4. Repeat. History tends to repeat itself. The fundemental concepts about data model that can be explored have been already explored. XML just brings a lot of old stuff back. To my current knowledge graph database is another example of repeating old things. The paper is right about predicting the use of XML: (to my expereince) nowadays XMLs are used more for things like config files, while JSON(a more light-weight format) is the primary format used to pass the data.
5. Flexible schema. The notion of schema-last/unstructured data turns out to be popular in some sense, indicated by the rise of NoSQL systems. I only read the good things about flexible schema, I definitely need to learn more about their downsides comprehensively.
   



