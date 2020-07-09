
# Vanilla Cluster (3 node cluster)

``Vanilla Cluster``: three node CrateDB cluster that runs on a single host to provide 
parallel processing power on large scale data, when you only have one host.

## Contents

   - **update-dist**: script to install **CrateDB** locally. This script will install 
     the latest, unreleased, **CrateDB** under **dist/**, creating a link 
     **./crate -> dist/crate..**.
   - **dist**: installed **CrateDB** distribution.
   - **crate**: a symlink to the installed distribution in the *dist* folder, where
     you will also find a **crate-clone** git repository.
   - **conf**: **CrateDB** configurations, each node in the cluster has a folder
     in there, with the **crate.yml** and **log4j2.properties** configuration files.
   - **data**: **CrateDB** the nodes will persist their data under ``data/n<i>/nodes/0``.
   - **start-node**: script to start **CrateDB** with a given configuration specified
     as a node name, `e.g. n1`, in the parameters to the script.
   - **detach-from-cluster**: script to detach a node from the ``Vanilla Cluster```.
   - **unsafe-bootstrap**: script to bootstrap a node to form a new cluster. Which
     means, recreating its cluster state so that it may be started on its own.
   - **data.py**: python3 script produce sample data.

## start-node

   - **./start-node n1**
   - **./start-node n2**
   - **./start-node n3**

   Which will form the ``Vanilla Cluster``, electing a master. You can interact with the 
   ``Vanilla Cluster`` by opening a browser and pointing it to *http://localhost:4200*, 
   *CrateDB*'s `Admin UI`_.

## Sample table for data.py

::

  CREATE TABLE IF NOT EXISTS "doc"."logs" (
     "log_time" TIMESTAMP WITH TIME ZONE NOT NULL,
     "client_ip" IP NOT NULL,
     "request" TEXT NOT NULL,
     "status_code" SMALLINT NOT NULL,
     "object_size" BIGINT NOT NULL
  )
  CLUSTERED INTO 4 SHARDS
  WITH (
     number_of_replicas = '0-1',
  );

We have a default min number of replicas of zero, and a max of one for each of our four 
shards. A replica is simply a copy of a shard.


## Downscaling

1. We need to ensure that the number of replicas matches the number of nodes:

::

  ALTER TABLE logs SET (number_of_replicas = '1-all');

2. After replication is completed, we can take down all the nodes in the cluster
   (**ctrl^C** in the terminal).

3. Run **./detach-from-cluster ni**, where i in [2,3], to detach **n2** and **n3** from the cluster.
   We will let **n1** form a new cluster all by itself, with access to the original data.

4. Change **n1**'s configuration **crate.yml**. The best practice is to select the node
   that was master, as then we know it had the latest version of the cluster state. For
   us, since we are running in a single host the cluster state is more or less
   guaranteed to be consistent across all nodes. In principle, however, the cluster could
   be running across multiple hosts, and then we would want the master node to become the
   new single node cluster:

   ::

     cluster.name: simple   # don't need to change this
     node.name: n1
     stats.service.interval: 0
     network.host: _local_
     node.max_local_storage_nodes: 1

     http.cors.enabled: true
     http.cors.allow-origin: "*"

     transport.tcp.port: 4301
     #gateway.expected_nodes: 3
     #gateway.recover_after_nodes: 2
     #discovery.seed_hosts:
     #  - 127.0.0.1:4301
     #  - 127.0.0.1:4302
     #cluster.initial_master_nodes:
     #  - 127.0.0.1:4301
     #  - 127.0.0.1:4302

5. Run *./unsafe-bootstrap n1** to let **n1** join a new cluster when it starts.

6. Run **./start-node n1**.
   Panic not, the cluster state is *[YELLOW]*, we sort that out with:

   ::

     ALTER TABLE logs SET (number_of_replicas = '0-1');

.. _`Admin UI`: http://localhost:4200
