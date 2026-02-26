###  General docker based instruction


Примерная структура:
```bash
      etcd (DCS)
      |
 ---------
 |       |
pg1     pg2
(master)(replica)
   |
 logical → analytics / backup / BI
```

```bash
mkdir -p patroni-lab{data1,data2}
cd patroni-lab
sudo chown -R 999:999 data1 data2
```

`docker-compose.yml`

```yaml
services:

  etcd:
    image: quay.io/coreos/etcd:v3.5.12
    container_name: etcd
    command: >
      /usr/local/bin/etcd
      --enable-v2=true
      --name etcd0
      --data-dir /etcd-data
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://etcd:2379
      --listen-peer-urls http://0.0.0.0:2380
      --initial-advertise-peer-urls http://etcd:2380
      --initial-cluster etcd0=http://etcd:2380
      --initial-cluster-state new
    volumes:
      - ./etcd-data:/etcd-data
    networks:
      - patroni-net


  pg1:
    image: d2cio/patroni:latest
    container_name: pg1
    environment:
      PATRONI_NAME: pg1
      PATRONI_SCOPE: labcluster

      PATRONI_ETCD_HOSTS: "['etcd:2379']"

      PATRONI_RESTAPI_LISTEN: 0.0.0.0:8008
      PATRONI_RESTAPI_CONNECT_ADDRESS: pg1:8008

      PATRONI_POSTGRESQL_LISTEN: 0.0.0.0:5432
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: pg1:5432
      PATRONI_POSTGRESQL_DATA_DIR: /data/pgdata

      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: postgres

      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: repl123

    volumes:
      - ./data1:/data
    depends_on:
      - etcd
    networks:
      - patroni-net


  pg2:
    image:
    container_name: pg2
    environment: d2cio/patroni:latest
      PATRONI_NAME: pg2
      PATRONI_SCOPE: labcluster

      PATRONI_ETCD_HOSTS: "['etcd:2379']"

      PATRONI_RESTAPI_LISTEN: 0.0.0.0:8008
      PATRONI_RESTAPI_CONNECT_ADDRESS: pg2:8008

      PATRONI_POSTGRESQL_LISTEN: 0.0.0.0:5432
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: pg2:5432
      PATRONI_POSTGRESQL_DATA_DIR: /data/pgdata

      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: postgres

      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: repl123

    volumes:
      - ./data2:/data
    depends_on:
      - etcd
    networks:
      - patroni-net


networks:
  patroni-net:
    driver: bridge
```


```bash
docker exec -it pg1 bash
patronictl list

docker exec -it pg1 patronictl list

```


```bash
docker exec -it pg1 patronictl list
+------------+--------+------------+--------+---------+----+-----------+
|  Cluster   | Member |    Host    |  Role  |  State  | TL | Lag in MB |
+------------+--------+------------+--------+---------+----+-----------+
| labcluster |  pg1   | 172.24.0.4 |        | running |  1 |         0 |
| labcluster |  pg2   | 172.24.0.3 | Leader | running |  1 |         0 |
+------------+--------+------------+--------+---------+----+-----------+
```
```bash
docker stop pg2
docker exec -it pg1 patronictl list
+------------+--------+------------+--------+---------+----+-----------+
|  Cluster   | Member |    Host    |  Role  |  State  | TL | Lag in MB |
+------------+--------+------------+--------+---------+----+-----------+
| labcluster |  pg1   | 172.24.0.4 | Leader | running |  2 |         0 |
| labcluster |  pg2   | 172.24.0.3 |        | stopped |    |   unknown |
+------------+--------+------------+--------+---------+----+-----------+
```

> Patroni через DCS (etcd) хранит информацию о лидере. Если heartbeat пропадает, остальные ноды проводят election и одна из них вызывает pg_promote.

- ❓ Как происходит восстановление ноды?
- Patroni автоматически подключает ноду к новому лидеру, используя pg_rewind или basebackup, синхронизирует timeline и переводит в replica.

- ❓ Что если старый мастер вернулся?
- Он не становится мастером, а автоматически переинициализируется как реплика.

- ❓ Что если сеть порвалась?
- Лидерство определяется через DCS (etcd), split-brain невозможен без кворума.



