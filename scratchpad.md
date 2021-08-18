## Connecting

## Sentinel

```sh
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis-test-cluster -o jsonpath="{.data.redis-password}" | base64 --decode)
kubectl run --namespace default redis-test-cluster-client --rm --tty -i --restart='Never' \
    --env REDIS_PASSWORD=$REDIS_PASSWORD \
    -l role=client \
    --image docker.io/bitnami/redis:6.2.1-debian-10-r36 -- bash

redis-cli -h redis-test-cluster -p 6379 -a $REDIS_PASSWORD # Read only operations
redis-cli -h redis-test-cluster -p 26379 -a $REDIS_PASSWORD # Sentinel access
```

## Master/Slave mode
```sh
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis-test-cluster -o jsonpath="{.data.redis-password}" | base64 --decode)
kubectl run --namespace default redis-test-cluster-client --rm --tty -i --restart='Never' \
    --env REDIS_PASSWORD=$REDIS_PASSWORD \
   --image docker.io/redis:local -- bash

kubectl run --namespace default redis-test-cluster-client --rm --tty -i --restart='Never' \
    --env REDIS_PASSWORD=$REDIS_PASSWORD \
   --image docker.io/bitnami/redis:6.2.1-debian-10-r36 \
   -l role=client \
   -- bash

redis-cli -h redis-test-cluster-master -a $REDIS_PASSWORD
redis-cli -h redis-test-cluster-slave -a $REDIS_PASSWORD

kubectl port-forward --namespace default svc/redis-test-cluster-master 6379:6379 &
    redis-cli -h 127.0.0.1 -p 6379 -a $REDIS_PASSWORD
```

## Network policy blocking all traffic

### Slave

```sh
$ time redis-cli ping
PONG

real	0m16.402s
user	0m0.005s
sys	0m0.003s

$ redis-cli role
1) "slave"
2) "redis-test-cluster-master-0.redis-test-cluster-headless.default.svc.cluster.local"
3) (integer) 6379
4) "connect"
5) (integer) -1

$ redis-cli info
role:slave
master_host:redis-test-cluster-master-0.redis-test-cluster-headless.default.svc.cluster.local
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:508
master_link_down_since_seconds:335
slave_priority:100
slave_read_only:1
connected_slaves:0
master_failover_state:no-failover
master_replid:b20ba4364419eab0fd13971e5e401dc6d6d3d0cf
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:508
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:508
```

### Master

```sh
$ time redis-cli ping
PONG

real	0m0.010s
user	0m0.009s
sys	0m0.001s

$ redis-cli role
1) "master"
2) (integer) 508
3) (empty array)

$ redis-cli info
# Replication
role:master
connected_slaves:0
master_failover_state:no-failover
master_replid:b20ba4364419eab0fd13971e5e401dc6d6d3d0cf
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:508
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:508
```

### Sentinel Write

https://lzone.de/cheat-sheet/Redis%20Sentinel

```sh
```

### Sentinel RO

```sh
```

- The node is starting up (not ready)

Ping seems to fail until redis can connect the first time.

- The node is in standalone mode (not ready)

Ping seems to fail until redis can connect the first time.

- The node is syncing itself with the cluster (not ready)

>> Is it OK to wait until 'master_link_status' becomes 'up', and 'master_sync_in_progress' becomes '0' and 'master_last_io_seconds' becomes >= 0?
> If you have no reason to believe something has gone haywire, this ought to tell you that the initial sync process has completed, yes.
https://groups.google.com/g/redis-db/c/JPvnyfUWx_Q?pli=1

```sh
while :; do echo '### INFO'; kubectl exec redis-test-cluster-node-2 -c redis -- 2>/dev/null redis-cli -h localhost -p 6379 -a $REDIS_PASSWORD info | grep 'master_link_status\|master_sync_in_progress\|master_last_io_seconds' ;done
### INFO
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:1
### INFO
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
```

- The node is syncing its state (as a donor) to another node (ready)

master == ready

- The node is up and running (ready)
