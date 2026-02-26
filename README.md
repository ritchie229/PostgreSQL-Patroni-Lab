### PATRONI LAB

This is a demo of Patroni lab consisting of two identical containers running PostgreSQL+Patroni and one container with etcd orchestrating them. PGDATA's are initiated on docker volumes.

```bash
etcd
  |
patroni1 ─ postgres
patroni2 ─ postgres
```
- patroni_docker.md - All deployment instruction (https://github.com/ritchie229/PostgreSQL-Patroni-Lab/blob/main/patroni_docker.md)
- patroni_baremetal.md - Relative instruction on deploying on baremetal (https://github.com/ritchie229/PostgreSQL-Patroni-Lab/blob/main/patroni_baremetal.md)

### SOME ANNOTATIONS TO REMEMBER

> [!NOTE]
> Patroni — это открытое решение для управления отказоустойчивыми кластерами PostgreSQL (High-Availability), которое обеспечивает автоматическую репликацию, мониторинг и автоматическое переключение (failover) в случае падения главного узла. Его часто используют в продакшн-окружениях (в том числе bare-metal и в облаке), где нужна непрерывность работы PostgreSQL.


**Организацию кластерной логики PostgreSQL**
- Следит за состоянием всех узлов в кластере.
- Обеспечивает выбор главного (primary/master) и поддерживает остальных как реплик (standby/replica).

**Автоматический failover.** Если главный узел перестаёт отвечать, Patroni:
- Определяет факт сбоя.
- Выбирает нового кандидата на роль primary (на основе консенсуса/стратегии).
- Переключает репликацию так, чтобы новый primary начал принимать записи.

**Репликация**
-Работает с асинхронной или синхронной репликацией PostgreSQL. Реплики получают WAL-записи и держат данные в согласованном (консистентном) состоянии.

> Patroni — это Python-демон, который запускается на каждом узле и использует несколько ключевых механизмов:
1. **DCS** (Distributed Configuration Store) - Patroni использует распределённое хранилище конфигурации для синхронизации статуса всех узлов и выбора лидера:
- etcd
- Consul
- ZooKeeper

В DCS хранится:
- Информация о текущем лидере (primary)
- Текущий статус узлов
- Ключ для организации выборов

Это не репозитории данных PostgreSQL — это лишь координационный слой.

> **Хотя есть и варианты, в основном etcd разворачивают на всех нодах, чтобы обеспечить кворум DCS без выделенного кластера и получить отказоустойчивость при минимальных затратах.**



2. **Мониторинг статуса PostgreSQL.** Patroni регулярно опрашивает:
- Работает ли PostgreSQL
- Может ли узел быть primary
- Может ли узел быть репликой
Если узел не отвечает, Patroni инициирует автоматические действия.

3. **Failover / Switchover**
- Failover - Автоматический переход на нового primary при падении старого.
- Switchover - Управляемая (ручная) смена primary, например для обновлений или обслуживания.

4. **Конфигурация** - загружаются с YAML-файлом конфигурации, в котором указываются:
- Адреса DCS
- Параметры репликации
- Правила выбора лидера
- Сети/порты

**Минимальный набор для запуска кластера:**
```
etcd (или Consul / ZooKeeper) → DCS
patroni.yml на каждом узле
PostgreSQL на каждом узле
Сеть между узлами
```


**Условия:**
- bi-directional connectivity
- identical or similar platforms
- similar performance
- identical PostgreSQL instance
Так как репликация физическая, требует бинарную совместимость.


**Обновление Postgres**

Minor (15.3 - 15.7):
- обновляем реплику, проверяем, промоутим
- обновляем мастера, ставшего репликой

Major (15 - 16)
- pg_upgrade
- logical replication
- blue/green cluster
