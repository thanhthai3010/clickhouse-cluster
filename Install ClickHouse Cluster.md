# I. ZooKeeper Cluster(Multi-Node) Installation:
1. Download Apache ZooKeeper:

```sh
$ wget http://mirror.downloadvn.com/apache/zookeeper/zookeeper-3.4.12/zookeeper-3.4.12.tar.gz
```

2. Extract, Move and Rename ZooKeeper Files:

```sh
$ sudo tar -xvf zookeeper-3.4.12.tar.gz
$ sudo mv zookeeper-3.4.12 /var/lib
$ cd /var/lib
$ sudo mv zookeeper-3.4.12 zookeeper
```

3. Create a Configuration file Under /zookeeper/conf

```sh
$ sudo nano zookeeper/conf/zoo.cfg
# Enter following configurations to zoo.cfg
# The length of a single tick, which is the basic time unit used by ZooKeeper, as 	measured in milliseconds. It is used to regulate heartbeats, and timeouts. For example, the minimum session timeout will be two ticks.
$ tickTime=2000
# The location where ZooKeeper will store the in-memory database snapshots and,	 unless specified otherwise, the transaction log of updates to the database.
$ dataDir=/disk2/zookeeper-data/
# The port to listen for client connections; that is, the port that clients attempt to connect to. Default port is 2181
$ clientPort=2181
# This is the timeout limit, which indicates the length of time for one of the zookeeper nodes in quorum have to connect to the leader
$ initLimit=5
# This specifies the limit on how much apart the individual nodes can be out-of-sync (i.e out-of-date) from the leader.
$ syncLimit=2
# The above two init and sync limit are calculated using tickTime. By default tickTime is set to 2000 in the zoo.cfg. This means 2000 milliseconds. So, when we set initLimit as 5, multiply that by tickTime to calculate it in seconds. So, initLimit=5*2000=10000=10 	seconds. syncLimit=2*2000=4000=4 seconds.
# server.1 → it is the zookeeper server id to specific only for that host
# zoo1 → it is the zookeeper server instance’s hostname or IP. In your cluster, you will enter IPs or hostnames of the machines which you would like to install zookeeper.
# Don’t change the “:2888:3888” that is at the end of the nodes. Zookeeper nodes will use these ports to connect the individual follower nodes to the leader nodes. The another port is used for leader election.
	#Zookeeper Server 1
	server.1=ch_node_1:2888:3888
	#Zookeeper Server 2	
    server.2=ch_node_2:2888:3888
	#Zookeeper Server 3		
    server.3=ch_node_3:2888:3888
# Save & Exit
$ Ctrl + X + Y -> Enter
```

4. Define Machine ID - Value must be unique for each machine and between 1 and 255

```sh
$ # on ch_node_1
$ # cd to zookeeper-data folder first
$ cd /disk2/zookeeper-data/
$ touch zookeeper/myid
$ echo "1" > zookeeper/myid 
$ cat zookeeper/myid
```

Do the same things with another machine ID to other nodes

```sh
$ # on ch_node_2
$ # cd to zookeeper-data folder first
$ cd /disk2/zookeeper-data/
$ touch zookeeper/myid
$ echo "2" > zookeeper/myid 
$ cat zookeeper/myid
```

```sh
$ # on ch_node_3
$ # cd to zookeeper-data folder first
$ cd /disk2/zookeeper-data/
$ touch zookeeper/myid
$ echo "3" > zookeeper/myid 
$ cat zookeeper/myid
```

5. Start ZooKeeper Instances
Perform on all nodes. The second instance you started is should be leader. Check statuses of nodes again, when you performed all start operations.

```sh
$ cd zookeeper/bin/
$ ./zkServer.sh start
$ ./zkServer.sh status
```

# II. Install ClickHouse Cluster:
1. Install ClickHouse follow this tutorial: 

```sh
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4    # optional

$ sudo apt-add-repository "deb http://repo.yandex.ru/clickhouse/deb/stable/ main/"
$ sudo apt-get update

$ sudo apt-get install -y clickhouse-server clickhouse-client

$ sudo service clickhouse-server start
```

2. Install table distributed in 3 shards and replicated twice.
![alt text](https://static1.squarespace.com/static/58d158119f745633ea326878/t/5af474bb0e2e72187f9cea80/1525970114389/concept.png?format=1000w "ClickHouse 3 nodes, 3 shards, repliaced twice concept")

3. Cluster Configuration:
Let’s start with a straightforward cluster configuration that defines 3 shards and 2 replicas. Since we have only 3 nodes to work with, we will setup replica hosts in a `Circle` manner meaning we will use the first and the second node for the first shard, the second and the third node for the second shard and the third and the first node for the third shard. Just like so:
- 1st shard, 1st replica, hostname: ch_node_1
- 1st shard, 2nd replica, hostname: ch_node_2
- 2nd shard, 1st replica, hostname: ch_node_2
- 2nd shard, 2nd replica, hostname: ch_node_3
- 3rd shard, 1st replica, hostname: ch_node_3
- 3rd shard, 2nd replica, hostname: ch_node_1

The configuration section may look like this:

```xml
<remote_servers>
   <perftest_3shards_3replicas>
      <shard>
         <replica>
            <internal_replication>true</internal_replication>
            <default_database>testcluster_shard_1</default_database>
            <host>ch_node_1</host>
            <port>9000</port>
         </replica>
         <replica>
            <default_database>testcluster_shard_1</default_database>
            <host>ch_node_2</host>
            <port>9000</port>
         </replica>
      </shard>
      <shard>
         <internal_replication>true</internal_replication>
         <replica>
            <default_database>testcluster_shard_2</default_database>
            <host>ch_node_2</host>
            <port>9000</port>
         </replica>
         <replica>
            <default_database>testcluster_shard_2</default_database>
            <host>ch_node_3</host>
            <port>9000</port>
         </replica>
      </shard>
      <shard>
         <internal_replication>true</internal_replication>
         <replica>
            <default_database>testcluster_shard_3</default_database>
            <host>ch_node_3</host>
            <port>9000</port>
         </replica>
         <replica>
            <default_database>testcluster_shard_3</default_database>
            <host>ch_node_1</host>
            <port>9000</port>
         </replica>
      </shard>
   </perftest_3shards_3replicas>
</remote_servers>
```

As you can see now we have the following storage schema:
- cluster_node_1 stores 1st shard, 1st replica and 3rd shard, 2nd replica
- cluster_node_2 stores 1st shard, 2nd replica and 2nd shard, 1st replica
- cluster_node_3 stores 2nd shard, 2nd replica and 3rd shard, 1st replica

4. Database Schema:
As discussed above, in order to separate shards between each other on the same node shard-specific databases are required.
- Schemas of the 1st Node
    + testcluster_shard_1
    + testcluster_shard_3
- Schemas of the 2nd Node
    + testcluster_shard_2
    + testcluster_shard_1
- Schemas of the 3rd Node
    + testcluster_shard_3
    + testcluster_shard_2
5. Replicated Table Schema:
To enable replication ZooKeeper is required. ClickHouse will take care of data consistency on all replicas and run restore procedure after failure automatically. It's recommended to deploy ZooKeeper cluster to separate servers. And then config `Zookeeper` cluster:

```xml
    <zookeeper>
        <node index="1">
            <host>ch_node_1</host>
            <port>2181</port>
        </node>
        <node index="2">
            <host>ch_node_2</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host>ch_node_3</host>
            <port>2181</port>
        </node>
    </zookeeper>
```

Now let’s setup replicated tables for shards. `ReplicatedMergeTree` table definition requires two important parameters:
- Table Shard path in `Zookeeper`
- Replica Tag
Zookeeper path should be unique for every shard, and Replica Tag should be unique within each particular shard:

```sql
### 1st Node:
CREATE TABLE testcluster_shard_1.replicated 
… 
Engine=ReplicatedMergeTree('/clickhouse/tables/tc_shard_1/{table_name}', '{replica_1_name}', …)
CREATE TABLE testcluster_shard_3.replicated 
… 
Engine=ReplicatedMergeTree('/clickhouse/tables/tc_shard_3/{table_name}', '{replica_2_name}', …)

### 2st Node:
CREATE TABLE testcluster_shard_2.replicated 
… 
Engine=ReplicatedMergeTree('/clickhouse/tables/tc_shard_2/{table_name}', '{replica_1_name}', …)
CREATE TABLE testcluster_shard_1.replicated 
… 
Engine=ReplicatedMergeTree('/clickhouse/tables/tc_shard_1/{table_name}', '{replica_2_name}', …)

### 3st Node:
CREATE TABLE testcluster_shard_3.replicated 
… 
Engine=ReplicatedMergeTree('/clickhouse/tables/tc_shard_3/{table_name}', '{replica_1_name}', …)
CREATE TABLE testcluster_shard_2.replicated 
… 
Engine=ReplicatedMergeTree('/clickhouse/tables/tc_shard_2/{table_name}', '{replica_2_name}', …)
```

6. Distributed Table Schema:
Distributed-table is actually a kind of "view" to local tables of ClickHouse cluster. `SELECT` query from a distributed table will be executed using resources of all cluster's shards.

```sql
CREATE TABLE distributed(date Date, id UInt32, shard_id UInt32)
    ENGINE = Distributed(perftest_3shards_3replicas, default, replicated, shard_id);
```

- `perftest_3shards_3replicas`: the name of cluster config.
- `default`: the `default` database.
- `replicated`: the `replicated` table.
- `shard_id`: the sharding key.
When query to the distributed table comes, ClickHouse automatically adds corresponding default database for every local `replicated` table.