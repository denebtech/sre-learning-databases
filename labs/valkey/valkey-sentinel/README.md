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

```bash
sudo systemctl restart valkey-server
sudo systemctl status valkey-server
```

Check that Valkey is listening on the network interface and not just localhost:

```bash
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

```bash
sudo systemctl restart valkey-server
```

Verify replication:

```bash
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

```bash
sudo systemctl start valkey-sentinel
sudo systemctl status valkey-sentinel
```

## Validate the cluster

From any node:

```bash
valkey-cli -p 26379 SENTINEL masters
valkey-cli -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
```

```sh
vagrant@valkey-node-15:~$ valkey-cli -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
1) "192.168.56.15"
2) "6379"
```

Expected result:

```text
192.168.56.15
6379
```

You can also inspect the Sentinel view from each node:

```bash
valkey-cli -h 192.168.56.15 -p 26379 SENTINEL sentinels valkeySentinel
valkey-cli -h 192.168.56.16 -p 26379 SENTINEL sentinels valkeySentinel
valkey-cli -h 192.168.56.17 -p 26379 SENTINEL sentinels valkeySentinel
```

## Failover test

To test automatic failover, stop the master service on node1:

```bash
sudo systemctl stop valkey-server
```

Then check which node becomes the new master:

```bash
valkey-cli -p 26379 SENTINEL get-master-addr-by-name valkeySentinel
```

A quorum of 2 Sentinel nodes is required for a failover decision.

## Node IPs used in this lab

- node1: `192.168.56.15`
- node2: `192.168.56.16`
- node3: `192.168.56.17`

This lab is designed around a master/replica topology with a 3-node Sentinel quorum for high availability and automatic failover.
