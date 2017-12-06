---
layout: article
title:  "Understanding Vitess topology"
date:   2017-12-06 13:39:00 -0400
categories: 
---

Vitess has three topology implementations, Etcd, Zookeeper and Consul.
For the purpose of this exercise, I will talk about Consul.

Vitess mostly uses the topology server to store its metadata and to lock servers when it needs to.

[You should check the official topology docs](https://github.com/youtube/vitess/blob/master/doc/TopologyService.md).

The topology server is useful for service discovery too. So in environments like k8s/containers, we don't need to force IPs for machines, as the topology server holds the hosts' information.

Vitess has an abstract concept called the 'cell'. A cell could be a data center, an availability zone, and/or both. As it is an abstract concept, it is up to the infrastructure using Vitess to decide.
For this exercise, let's say we pick one cell per data center.

Vitess wants two topology cluster types, one global for cross-cell information, one local per cell.
The local server is the one that runs in the current cell.
The global server is one that will hold the global data.

Note: Vitess is still available even if the global topology service is down, it only limits us to do operations like a re-sharding.
The global and local topology clusters have non-overlapping paths. Thus, the same can be used as global and as a local topology server.
The local server has most of the global information replicated to it, therefore, Vitess processes will not reach the global server much during its lifetime. This 'replicated information' is made by Vitess, and do not require any special Consul configuration.
Vitess uses the Key/Value functionality from Consul to store data.

### Let's take an example of setup:

#### Servers running MySQL and vttablet:
```

- db1.east.arthurnn.com (master)
  + db1.replica1.east.arthurnn.com (replica)
  + db1.replica2.east.arthurnn.com (replica)
  + db1.replica1.west.arthurnn.com (replica)
  + db1.replica2.west.arthurnn.com (replica)
- db2.east.arthurnn.com (master)
  + db2.replica1.west.arthurnn.com (replica)
  + db2.replica1.east.arthurnn.com (replica)
  + db2.replica2.east.arthurnn.com (replica)
```
#### Consul clusters:
```

- consul.west.arthurnn.com
- consul.east.arthurnn.com
```

#### vtgate servers:
```

- vtgate.east.arthurnn.com
- vtgate.west.arthurnn.com
```

#### vtctld servers:
```

- vtctld.east.arthurnn.com
```

In that setup, we have two cells, west and east. Each cell needs a vtgate, so it can route traffic to a close by tablet when possible.
We also have a main vtctld to receive topology changes commands from its client(vtctlclient).
When we start the vtctld server, we point to the global topology server of our choice. Lets chose consul.east:
```
topology_flags="-topo_implementation consul -topo_global_server_address consul.east.arthurnn.com:8500 -topo_global_root vitess/global"

$VTROOT/bin/vtctld \
  $topology_flags \
  -logtostderr=true \
  -cell east \
  -web_dir $VTTOP/web/vtctld \
  -web_dir2 $VTTOP/web/vtctld2/app \
  -workflow_manager_init \
  -workflow_manager_use_election \
  -service_map 'grpc-vtctl' \
  -backup_storage_implementation file \
  -file_backup_storage_root $VTDATAROOT/backups \
  -port 80 \
  -grpc_port 15999 \
  -pid_file $VTDATAROOT/tmp/vtctld.pid
```

Then the local topology addresses are configured like this, for each cell:
```
vtctld_address=vtctld.east.arthurnn.com:15991

# creates the east cell
vtctlclient -server $vtctld_address AddCellInfo -server_address consul.east.arthurnn.com:8500 -root vitess/east east

# creates the west cell
vtctlclient -server $vtctld_address AddCellInfo -server_address consul.west.arthurnn.com:8500 -root vitess/west west
```

The command `AddCellInfo` is creating the cell information into the global consul cluster. The idea here is: When we start a vtgate, we tell vtgate where the global topology cluster is, therefore it can look-up, from the global server, what is the `CellInfo` for that cell, then it knows where the local topology server is. It will connect to it, and it will do most of the operations there, regarding locking, reading info about the `vttablet+MySQL` combos, etc. (check the vitess doc to a list of all operations).
Note we don't pass the global consul server to the `vtctlclient` command. We only pass the `-server` option, pointing to the `vtctld` process. The `vtctld` process knows about the global consul server, and it will write to it. Thus, there is no need to tell the global consul address in the `vtctlclient` command.

When we start a `vttablet`, we also need to point to the global topology server, so it can look-up the info about the local topology server.
From the example above, each db{1,2} hosts would start a `vttablet` passing the global topology cluster. `vttablet` will find the local topo cluster from the `CellInfo` and will cache where the vttablet is. It will also announce its presence to the local vtgates.
Each vtgate will boot-up pointing to the global topo cluster so it can also find the local topology cluster config, and it will know where each shard is, to route queries accordingly. When doing readonly(replica) queries vtgate will try to dispatch those to local tablets.

`vtctld` is an application/service that also connects to the global topology server. It is responsible for doing reads and writes into the topology. You can see that the `vtctlclient` command pointed to the `vtctld` server.

### Conclusion 

Vitess always wants the global topology cluster as an entry point, and from there it can lookup the local topology cluster and information about the keyspaces and shards. Thus, when we start commands like `vtctld`, `vtgate`, `vttablet` we always need to pass the global topology server.

Vitess extensively states that it does not use either of the topologies servers to serve queries, as it understands that in order to the topology server be consistent it has a cost writing and reading data to it.
