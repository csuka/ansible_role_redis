[![pipeline status](https://gitlab.com/kevin-ansible-roles/redis/badges/master/pipeline.svg)](https://gitlab.com/kevin-ansible-roles/rundeck/-/commits/master)

# Ansible Role: Redis

Installs [Redis](http://redis.io/) on EL 8, this role also includes configuring redis-sentinel.

 * v1.0.0 - initial release
 * v1.0.1 - add molecule testing
 * v1.0.2 - improved single host play and fixed IP and included unixsocket by default

## Requirement

Redis is memory exhaustive, so having one cpu core is sufficient.

Also, to write the the database to disk, Redis requires at least 2 times the memory it is using.

Meaning, if Redis consumes 4GB, it will need another 4GB of memory to write the db to disk. If there is only 2GB on the system available, the database will not be written to disk. So it's a good practice to ensure keys are cleaned up and Redis is configured to half of the system memory. Also keep in mind the system requires swap, which could be 2GB.

In short, if the a system wants 10GB of ram to be used for Redis, the system would require at least 22GB of ram.

## Role Variables

See `defaults/main.yml` for all variables.

When Redis is installed via remi's repo, version 6.0.x is installed. The EPEL repo is installed as a dependency of Remi's repo.
A file is placed in `/etc/dnf/modules.d/mysql.module`, to enable installation.

```yaml
redis_version: 6.0
redis_remi_repo: true
```

Interface to bind. By default, Redis already binds to `127.0.0.1`, this does not has to be set.

```yaml
redis_ip: 10.0.0.1
```

Redis also listens a local Unix socket.

```yaml
redis_unixsocket: '/var/run/redis/redis.sock'
```

Valid levels are `debug`, `verbose`, `notice`, and `warning`.

```yaml
redis_loglevel: "notice"
```

Set allowed memory, preferably in bytes. Set value `0` for unlimited.

```yaml
redis_maxmemory: 1073741824 # 1GB
```

Close a connection after a client is idle `N` seconds. Set to `0` to disable timeout.

```yaml
redis_timeout: 300
```

The number of Redis databases.

```yaml
redis_databases: 16
```

Snapshotting configuration; setting values in this list will save the database to disk if the given number of seconds (e.g. `900`) and the given number of write operations (e.g. `1`) have occurred.

```yaml
# Set to an empty list to disable persistence, this will save the DB to disk
redis_save:
  - 900 1
  - 300 10
  - 60 10000
```

Database compression and location configuration.

```yaml
redis_rdbcompression: "yes"
redis_dbfilename: dump.rdb
redis_dbdir: /var/lib/redis
```

The method to use to keep memory usage below the limit, if specified. See [using Redis as an LRU cache](http://redis.io/topics/lru-cache).

```yaml
redis_maxmemory_policy: "noeviction"
```

Number of samples to use to approximate LRU. See [Using Redis as an LRU cache](http://redis.io/topics/lru-cache).

```yaml
redis_maxmemory_samples: 5
```

The appendonly option, if enabled, affords better data durability guarantees, at the cost of slightly slower performance.

```yaml
redis_appendonly: "no"
```

Valid values are `always` (slower, safest), `everysec` (happy medium), or `no` (let the filesystem flush data when it wants, most risky).

```yaml
redis_appendfsync: "everysec"
```


Set a password to require authentication to Redis. You can generate a strong password using `echo "my_password_here" | sha256sum`.

```yaml
redis_requirepass: ""
```

For an added layer of security, you can disable certain Redis commands (this is especially important if Redis is publicly accessible).

```yaml
redis_disabled_commands: []
```

Example:
```yaml
redis_disabled_commands:
  - FLUSHDB
  - DEL
  - CONFIG
  - SHUTDOWN
```

To include variables which are not set in the config, copy the extra_commands list for the host var, and append them to the list:

```yaml
redis_extra_commands:
  replica-serve-stale-data: "yes"
  ...
  ...
  my_var: my_value
```

## Kernel settings

Redis requires several kernel settings to be changed, e.g. disabling THP.

The following settings are applied to the system in config and during runtime:

  * vm.overcommit_memory = 1
  * disables THP,
  * net.core.somaxconn = 511 (the redis tcp_backlog value)

## Warning message

After Redis starts, Redis warns in the log about setting valid Timeout settings in the service file. The values set in the service file are actually valid, the warning can be safely ignored.

## Redis values during runtime

A lot of Redis values can be set during runtime via redis-cli. Due to this design, Redis should not restart after a config change.
This role is setup so that Redis only will restart after installation.

If for some reason, e.g. testing purposes, you still want to restart redis after a config change when this role is executing, set:

```yaml
redis_restart_on_config_change: true
```

## Sentinel

When Redis is installed, the package redis-sentinel is installed as well. Enabling redis-sentinel requires at least 3 nodes, to reach the quorum.

To reduce costs, one could setup:

 * 1 redis master
 * 1 redis slave, of the master
 * 1 redis-sentinel acting as an arbiter, does not contain redis data

Adjust the values accordingly for each host:

```yaml
# host 1
redis_sentinel:
  enabled: true
  role: master
  id: 1301e0422ab440f5c8ad6dc5c31eccd34ad44871

# host 2
redis_sentinel:
  enabled: true
  role: slave
  id: 1301e0422ab440f5c8ad6dc5c31eccd34ad44872

# host 3
redis_sentinel:
  enabled: true
  role: arbiter
  id: 1301e0422ab440f5c8ad6dc5c31eccd34ad44873
```

Sentinel will update the configuration automatically with additional information about replicas (in order to retain the information in case of restart). The configuration is also rewritten every time a replica is promoted to master during a failover and every time a new Sentinel is discovered. The config file is only placed once, which is during the first run.

## Cluster

< todo >


## Example Playbook

```yaml
---
- hosts: all
  roles:
    - role: redis
```
