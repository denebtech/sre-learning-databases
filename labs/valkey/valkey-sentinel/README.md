# Vagrant Sentinel lab

## Deploy

```sh
vagrant up
```

## Playbook

```sh
ansible-playbook playbook.yml
```

## Important: configure Valkey for multi-node access

The default Valkey service is usually bound to `127.0.0.1:6379`, which is only valid for local access. For a 3-node cluster, each node must listen on the private Vagrant network and be reachable from the other nodes.

Update `/etc/valkey/valkey.conf` on each node to use:

```ini
bind 0.0.0.0
protected-mode no
port 6379
daemonize no
supervised systemd
dir /var/lib/valkey
appendonly yes
```

Then restart the service:

```sh
sudo systemctl restart valkey-server
sudo systemctl status valkey-server
```

Check that Valkey is listening on the network interface and not just localhost:

```sh
ss -tulpn | grep 6379
```

You should see a listener like `0.0.0.0:6379`.

## Configure the master and replicas

### Master node (node1 / 192.168.56.15)

Keep the default master configuration:

```ini
bind 0.0.0.0
protected-mode no
port 6379
daemonize no
supervised systemd
dir /var/lib/valkey
appendonly yes
```

### Replica nodes (node2 and node3)

On `node2` and `node3`, add:

```ini
bind 0.0.0.0
protected-mode no
port 6379
daemonize no
supervised systemd
dir /var/lib/valkey
replicaof 192.168.56.15 6379
appendonly yes
```

Then restart the services:

```sh
sudo systemctl restart valkey-server
```

Verify replication:

```sh
valkey-cli -h 192.168.56.16 -p 6379 INFO replication
valkey-cli -h 192.168.56.17 -p 6379 INFO replication
```

Look for:

```text
role:slave
master_host:192.168.56.15
master_port:6379
master_link_status:up
```

## Configure Sentinel

Stop current sentinel service

```sh
sudo systemctl status valkey-sentinel.service
sudo systemctl stop valkey-sentinel.service
```

```sh
vagrant@valkey-node-16:~$ sudo systemctl status valkey-sentinel.service 
● valkey-sentinel.service - Advanced key-value store
     Loaded: loaded (/lib/systemd/system/valkey-sentinel.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-05 10:18:17 UTC; 14h ago
       Docs: https://valkey.io/docs/,
             man:valkey-sentinel(1)
   Main PID: 457 (valkey-sentinel)
     Status: "Ready to accept connections"
      Tasks: 5 (limit: 2317)
     Memory: 4.3M
        CPU: 1min 7.916s
     CGroup: /system.slice/valkey-sentinel.service
             └─457 "/usr/bin/valkey-sentinel *:26379 [sentinel]"

Nov 05 10:18:17 valkey-node-16 systemd[1]: Starting valkey-sentinel.service - Advanced key-value store...
Nov 05 10:18:17 valkey-node-16 systemd[1]: Started valkey-sentinel.service - Advanced key-value store.
vagrant@valkey-node-16:~$ sudo systemctl stop valkey-sentinel.service 
```

Backup defaul config file

```sh
sudo cp -pv /etc/valkey/sentinel.conf{,.bak}
```

Edit `/etc/valkey/sentinel.conf` on each node with:

```sh
sudo vim /etc/valkey/sentinel.conf
```

NOTE: Replace all "mymaster" default name by "valkeySentinel"

```ini
port 26379
bind 0.0.0.0
protected-mode no
daemonize no
sentinel monitor valkeySentinel 192.168.56.15 6379 2
sentinel down-after-milliseconds valkeySentinel 5000
sentinel failover-timeout valkeySentinel 10000
sentinel parallel-syncs valkeySentinel 1
```

Start Sentinel on each node:

```sh
sudo systemctl start valkey-sentinel
sudo systemctl status valkey-sentinel
```

## Validate the cluster

From any node:

```sh
valkey-cli -p 26379 SENTINEL masters
valkey-cli -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
1) "192.168.56.15"
2) "6379"
```
From host:

```sh
valkey.cli -h 192.168.56.15 -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
```

Expected result:

```sh
192.168.56.15
6379
```

You can also inspect the Sentinel view from each node:

```sh
valkey-cli -h 192.168.56.15 -p 26379 SENTINEL sentinels valkeySentinel
valkey-cli -h 192.168.56.16 -p 26379 SENTINEL sentinels valkeySentinel
valkey-cli -h 192.168.56.17 -p 26379 SENTINEL sentinels valkeySentinel
```

From host:

```sh
valkey.cli -h 192.168.56.15 -p 26379 SENTINEL sentinels valkeySentinel
```

-----------------------------------------------------------------------------------

# Apendix A: Testing

## Test 1 - Failover

To test automatic failover, stop the master service on node1 (IP: 192.168.56.15):

```sh
sudo systemctl stop valkey-sentinel.service
```

Then check which node becomes the new master:

```sh
valkey-cli -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
```

From host:

```sh
valkey.cli -h 192.168.56.16 -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
valkey.cli -h 192.168.56.17 -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
```

In both cases, I get node3 (IP: 192.168.56.17) as a new master:

```sh
1) "192.168.56.17"
2) "6379"
```

Start node1 to verify the master:

```sh
sudo systemctl start valkey-sentinel.service
```

```sh
valkey.cli -h 192.168.56.15 -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
1) "192.168.56.17"
2) "6379"
```

Master persist in node3.

## Test 2 - Replication

To test replication, add data on the master node first:

```sh
valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:01 "hello-from-master"
valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:02 "replica-check"
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:01 "hello-from-master"
OK
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:02 "replica-check"
OK
```

Then verify the data is present on the replicas:

```sh
valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:01
valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:01

valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:02
valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:02
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:01
"hello-from-master"
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:01
"hello-from-master"
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:02
"replica-check"
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:02
"replica-check"

```

Now stop one replica node to simulate a temporary outage. For example, stop node2 (192.168.56.16):

```sh
vagrant ssh valkey-node-16
sudo systemctl stop valkey-server
```

From the master, add new data while the replica is down:

```sh
valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:03 "written-while-node2-offline"
valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:04 "another-update"
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:03 "written-while-node2-offline"
OK
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.15 -p 6379 SET lab:replication:04 "another-update"
OK
```

Check that the remaining replica is updated, while node2 is still stopped:

```sh
valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:03
valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:04
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:03
"written-while-node2-offline"
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.17 -p 6379 GET lab:replication:04
"another-update"
```

Restart the stopped node and confirm it catches up with the master:

```sh
vagrant ssh valkey-node-16
sudo systemctl start valkey-server
sudo systemctl status valkey-server
```

Check replication status and validate the missing data is synchronized back to the restarted node:

```sh
valkey-cli -h 192.168.56.16 -p 6379 INFO replication
valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:03
valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:04
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.16 -p 6379 INFO replication
# Replication
role:slave
master_host:192.168.56.15
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:4926800
slave_repl_offset:4926800
replicas_repl_buffer_size:0
replicas_repl_buffer_peak:0
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
replicas_waiting_psync:0
master_failover_state:no-failover
master_replid:4017e6692fe104a0551b1373cd00131b37c736a9
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:4926800
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:10485760
repl_backlog_first_byte_offset:4922498
repl_backlog_histlen:4303
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:03
"written-while-node2-offline"
vagrant@valkey-node-15:~$ valkey-cli -h 192.168.56.16 -p 6379 GET lab:replication:04
"another-update"
```
