### 1️⃣ Про репликацию

> В Postgres есть физическая и логическая.
> Физическая — WAL streaming, бинарная, для HA.
> Логическая — публикации/подписки, для миграций и интеграций.

### 2️⃣ Про слоты

> eplication slot — механизм, который не даёт серверу удалить WAL, пока реплика его не получила.
> Может привести к разрастанию WAL, если реплика умерла.

### 3️⃣ Про failover

> При падении мастера — promote standby.
> Вручную: pg_promote().
> Автоматически — Patroni.

### 4️⃣ Про Patroni

> Patroni — менеджер HA для Postgres.
> Использует DCS (etcd/consul/k8s) для leader election.
> Сам управляет promote, rewind, slots.

### 5️⃣ Про backup

> pg_basebackup, WAL archive, PITR.
