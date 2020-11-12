<h1 align="center">Redis HA with Load Balancing, Corosync and Pacemaker</h1>
<br />

<div align="center">
  <a href="https://travis-ci.com/mariancraciun1983/ansible-redis-ha">
    <img src="https://travis-ci.com/mariancraciun1983/ansible-redis-ha.svg?branch=master" alt="Build Status" />
  </a>
  <a href="https://galaxy.ansible.com/mariancraciun1983/redis_ha">
    <img src="https://img.shields.io/ansible/role/51765" alt="Ansible Galaxy" />
  </a>
  <a href="https://galaxy.ansible.com/mariancraciun1983/redis_ha">
    <img src="https://img.shields.io/ansible/quality/51765" alt="Ansible Quality Score" />
  </a>
  <a href="https://opensource.org/licenses/MIT">
    <img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="MIT License" />
  </a>
</div>
<br />



## Introduction

  Ansible role to configure Redis, Sentinel with High Availability
    and automated load balancing with HAProxy, Corosync and Pacemaker


## Automated replication and failover

The base functionality offer by the role is to configure the Redis cluster with replication monitored
by Sentinel. 
You will need to have, ideally, 3 nodes, but 2 work ok too by adjusting the quorum size.

Each node should have the `redis_role` set to `master` (exactly 1)  and `slave` (2 or more)
```yaml
  host_vars:
    node1:
      redis_role: master
    node2:
      redis_role: slave
    node3:
      redis_role: slave
```

The role will work out of the box, with no configuration needed. Check [defaults/main.yml](./defaults/main.yml) for optional config vars

## Load Balancing and Automated Failover

Redis sentinel will ensure that the replication remains functional by choosing a new master and adjusting the replication slave targets. 

In order for the clients to know which is the master, which is the slave, they will need to connect first
to the sentinel, ask for the current master, than connect to it.

The approach found in this role takes advantage of the Corosync/Pacemaker to manage iptables rules, by automatically blocking writes on the slaves whenever Sentinel triggers failover events.

- each node will listen on 3 ports 6379 (default), 6380(RO), 6381 (RW), last 2 via iptables prerouting
- the master will have `redis_role=primary` corosync note attribute and `redis_role=replica` for slaves respectivelly
- corosync will block port 6381 when redis_role=replica
- corosync will block port 6381 and 6380 when redis_role=fail
- the load balancer will try 6381 port for RW and only one server will NOT reject the connections `redis_role=primary`

### Requirements
The nodes should have Corosync with Pacemaker already configured. Check my [corosync_pacemaker ansible role](https://github.com/mariancraciun1983/ansible-corosync-pacemaker) . Having `symmetric-cluster` is also required so that not all nodes will get the resources assigned to them, but only based on pacemaker colocation rules.

`playbook.yml`:

```yaml
- name: Prepare Corosync/Pacemaker
  hosts: all
  gather_facts: true
  roles:
    - mariancraciun1983.corosync_pacemaker
    - mariancraciun1983.redis_ha
```

group_vars/all.yml
```yaml
# corosync config
install_python3: true
corosync_hacluster_password: 1q2w3e4r5t
corosync_cluster_settings:
  - key: stonith-enabled
    value: "false"
  - key: no-quorum-policy
    value: ignore
  - key: start-failure-is-fatal
    value: "false"
  - key: symmetric-cluster
    value: "false"

redis_pacemaker: true
# this must be true only once, initially, when the Corosync node attributes need to be configured
# after that, redis sentinel will be triggering node attributes updates in case of a failover
redis_pacemaker_helpers_init: true
```

Running `crm_mon -AnfroRtc` will gives us the following:
```
Cluster Summary:
  * Stack: corosync
  * Current DC: node3 (3) (version 2.0.3-4b1f869f0f) - partition with quorum
  * Last updated: Thu Nov 12 10:24:07 2020
  * Last change:  Thu Nov 12 10:23:45 2020 by redis via crm_attribute on node3
  * 3 nodes configured
  * 6 resource instances configured

Node List:
  * Node node1 (1): online:
    * Resources:
  * Node node2 (2): online:
    * Resources:
      * RedisLBWriteBlock       (ocf::heartbeat:command_raw):    Started
  * Node node3 (3): online:
    * Resources:
      * RedisLBWriteBlock       (ocf::heartbeat:command_raw):    Started

Inactive Resources:
  * Clone Set: RedisLBReadBlock-clone [RedisLBReadBlock]:
    * RedisLBReadBlock  (ocf::heartbeat:command_raw):    Stopped
    * RedisLBReadBlock  (ocf::heartbeat:command_raw):    Stopped
    * RedisLBReadBlock  (ocf::heartbeat:command_raw):    Stopped
    * Stopped: [ node1 node2 node3 ]
  * Clone Set: RedisLBWriteBlock-clone [RedisLBWriteBlock]:
    * RedisLBWriteBlock (ocf::heartbeat:command_raw):    Started node2
    * RedisLBWriteBlock (ocf::heartbeat:command_raw):    Started node3
    * RedisLBWriteBlock (ocf::heartbeat:command_raw):    Stopped
    * Started: [ node2 node3 ]
    * Stopped: [ node1 ]

Node Attributes:
  * Node: node1 (1):
    * redis_role                        : primary   
  * Node: node2 (2):
    * redis_role                        : replica   
  * Node: node3 (3):
    * redis_role                        : replica   

Operations:
  * Node: node1 (1):
  * Node: node2 (2):
    * RedisLBWriteBlock: migration-threshold=1000000:
      * (12) start: last-rc-change="Thu Nov 12 10:23:45 2020" last-run="Thu Nov 12 10:23:45 2020" exec-time="31ms" queue-time="0ms" rc=0 (ok)
      * (13) monitor: interval="10000ms" last-rc-change="Thu Nov 12 10:23:45 2020" exec-time="12ms" queue-time="0ms" rc=0 (ok)
  * Node: node3 (3):
    * RedisLBWriteBlock: migration-threshold=1000000:
      * (12) start: last-rc-change="Thu Nov 12 10:23:45 2020" last-run="Thu Nov 12 10:23:45 2020" exec-time="18ms" queue-time="0ms" rc=0 (ok)
      * (13) monitor: interval="10000ms" last-rc-change="Thu Nov 12 10:23:45 2020" exec-time="16ms" queue-time="0ms" rc=0 (ok)

```

## Automated backups

To configure automated backups, use the following

```yaml
redis_backups: yes
redis_backups_cron:
  day: "*"
  hour: 4
  destination: /mnt/storage/backups/redis/
```

This will enable automated on the master server with redis-cli `--rdb` command. 
For more advanced/larger backups `rdiff-backup` could be a better candidate.



## Testing

Molecule with docker is being used with 2 scenarios:
 - default - non-pacemaker
 - corosync/pacemaker based

Running the tests:
```bash
pipenv install
pipenv run molecule test
```

## License

MIT License

The code contains the [iptables_raw](https://github.com/Nordeus/ansible_iptables_raw) ansible module which is also licensed under MIT License.