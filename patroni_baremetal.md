
#### Patroni on baremetal:
- OS: Rocky / CentOS / Alma / RHEL / Ubuntu (разница минимальна)
- DBMS: PostgreSQL 15
- DCS: etcd (самый частый вариант)

#### Три сервера:
- pg1 — 10.0.0.1
- pg2 — 10.0.0.2
- pg3 — 10.0.0.3

#### Схема:
```
[ pg1 ]----\
[ pg2 ]----- > etcd cluster <----- Patroni
[ pg3 ]----/
```
#### Patroni ⇄ etcd ⇄ PostgreSQL
- etcd = “мозг”
- Patroni = “менеджер”
- Postgres = “двигатель”

```bash
dnf install -y postgresql15-server postgresql15
apt install -y postgresql-15
```
> ❗️НЕ инициализируем БД вручную! ❌`postgresql-setup initdb`


### ETCD INSTALL
```bash
dnf install -y etcd
apt install -y etcd
```

#### Конфиг pg1 /etc/etcd/etcd.conf
```ini
ETCD_NAME="pg1"
ETCD_DATA_DIR="/var/lib/etcd"

ETCD_LISTEN_PEER_URLS="http://10.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.1:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.1:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.1:2379"

ETCD_INITIAL_CLUSTER="pg1=http://10.0.0.1:2380,pg2=http://10.0.0.2:2380,pg3=http://10.0.0.3:2380"

ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-cluster"
```
```bash
systemctl enable --now etcd
etcdctl member list
```

### PATRONI INSTALL (исп Ansible)

```bash
dnf install -y python3-pip && pip3 install patroni[etcd] psycopg2-binary
apt install -y python3-pip && pip3 install patroni[etcd]
```

#### Каталоги:
```bash
mkdir -p /data/patroni
mkdir -p /data/pgdata

chown -R postgres:postgres /data
```

#### Конфиг pg1 Patroni /etc/patroni.yml
```yaml
scope: mycluster
name: pg1

restapi:
  listen: 10.0.0.1:8008
  connect_address: 10.0.0.1:8008

etcd:
  hosts:
    - 10.0.0.1:2379
    - 10.0.0.2:2379
    - 10.0.0.3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10

    postgresql:
      use_pg_rewind: true
      use_slots: true

      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10

  initdb:
    - encoding: UTF8
    - data-checksums

  users:
    replicator:
      password: replpass
      options:
        - replication

postgresql:
  listen: 10.0.0.1:5432
  connect_address: 10.0.0.1:5432

  data_dir: /data/pgdata
  bin_dir: /usr/pgsql-15/bin

  authentication:
    superuser:
      username: postgres
      password: superpass

    replication:
      username: replicator
      password: replpass

  parameters:
    unix_socket_directories: '/var/run/postgresql'

  pg_hba:
    - host replication replicator 10.0.0.0/24 md5
    - host all all 10.0.0.0/24 md5

  create_replica_methods:
    - basebackup

  basebackup:
    checkpoint: fast

watchdog:
  mode: off

tags:
  nofailover: false
  noloadbalance: false
```

#### На pg2 и pg3 меняем только:
```yaml
name:
listen:
connect_address:
```


### ДАЛЕЕ:
```bash
systemctl disable --now postgresql 	# Отключаем стандартный PostgreSQL

su - postgres
patroni /etc/patroni.yml		# Запуск Patroni От postgres-пользователя:

patronictl list

+ Cluster: mycluster --------+
| Member | Role   | State   |
+--------+--------+---------+
| pg1    | Leader | running |
| pg2    | Replica| running |
| pg3    | Replica| running |

```
### Patroni system.d
```bash
/etc/systemd/system/patroni.service
```
```ini
[Unit]
Description=Patroni HA PostgreSQL
After=network.target

[Service]
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
Restart=always
LimitNOFILE=102400

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now patroni
```








