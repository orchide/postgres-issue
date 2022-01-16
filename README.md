# postgres-issue



# Start by installing kubedb like so 
- Get License [here](https://license-issuer.appscode.com/?p=kubedb-community)
- [Install Kubedb](https://kubedb.com/docs/v2021.12.21/setup/install/community/)


# Postgres config file (hot-postgres-conf)

>archive_command = '/bin/true'
archive_mode = on
archive_timeout = 1800s
bg_mon.history_buckets = 120
bg_mon.listen_address = '0.0.0.0'
autovacuum_analyze_scale_factor = 0.02
autovacuum_max_workers = 5
autovacuum_vacuum_scale_factor = 0.05
checkpoint_completion_target = 0.9
default_statistics_target = 100
effective_cache_size = 18GB
effective_io_concurrency = 200
extwlist.custom_path = '/scripts'
extwlist.extensions = 'btree_gin,btree_gist,citext,hstore,intarray,ltree,pgcrypto,pgq,pg_trgm,postgres_fdw,tablefunc,uuid-ossp,hypopg,timescaledb,pg_partman'
hot_standby = on
idle_in_transaction_session_timeout = 30000
log_autovacuum_min_duration = 0
log_checkpoints = on
log_connections = on
log_destination = csvlog
log_directory = '../pg_log'
log_disconnections = on
log_file_mode = 0644
log_filename = 'postgresql-%u.log'
log_line_prefix = '%t [%p]: [%l-1] %c %x %d %u %a %h '
log_lock_waits = on
log_min_duration_statement = 500
log_rotation_age = 1d
log_statement = none
log_temp_files = 0
log_truncate_on_rotation = on
logging_collector = on
maintenance_work_mem = 2GB
max_connections = 20000
max_locks_per_transaction = 64
max_parallel_maintenance_workers = 4
max_parallel_workers = 64
max_parallel_workers_per_gather = 4
max_prepared_transactions = 0
max_replication_slots = 10
max_standby_archive_delay = -1
max_standby_streaming_delay = -1
max_wal_senders = 10
max_wal_size = 4GB
max_worker_processes = 64
min_wal_size = 1GB
pg_stat_statements.track_utility = off
random_page_cost = 1.1
shared_buffers = 6GB
statement_timeout = 300000
tcp_keepalives_idle = 900
tcp_keepalives_interval = 100
track_commit_timestamp = off
track_functions = all
wal_buffers = 16MB
wal_keep_size = 8
wal_level = replica
wal_log_hints = on
>work_mem = 32MB




# Install config secret
```
kubectl create secret generic -n demo hot-postgres-conf --from-file= path/to/configfile
```

## Install postgres


```sh
apiVersion: kubedb.com/v1alpha2
kind: Postgres
metadata:
  name: hot-postgres
  namespace: data
spec:
  version: "13.2"
  configSecret:
    name: hot-postgres-conf
  replicas: 3
  standbyMode: Hot
  storageType: Durable
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi
  terminationPolicy: Delete
```

# exec into postgres pod and create irembo_user as SuperUser

```
CREATE USER irembo_user WITH LOGIN SUPERUSER PASSWORD 'password';
CREATE DATABASE irembo_user ;
```

# Install pgpool

````
kind: Postgres
metadata:
  name: hot-postgres
  namespace: data
spec:
  version: "13.2"
  configSecret:
    name: hot-postgres-conf
  replicas: 3
  standbyMode: Hot
  storageType: Durable
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi
  terminationPolicy: Delete
[root@control-plane ~]# cat /usr/local/bin/pgpool.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgpool
  namespace: data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pgpool
  template:
    metadata:
      labels:
        app: pgpool
    spec:
      containers:
      - name: pgpool
        image: pgpool/pgpool
        resources:
          requests:
            memory: "250Mi"
            cpu: "100m"
          limits:
            memory: "500Mi"
            cpu: "200m"
        env:
        - name: PGPOOL_PARAMS_BACKEND_HOSTNAME0
          value: "hot-postgres"
        - name: PGPOOL_PARAMS_BACKEND_PORT0
          value: "5432"
        - name: PGPOOL_PARAMS_BACKEND_WEIGHT0
          value: "1"
        - name: PGPOOL_PARAMS_BACKEND_FLAG0
          value: "ALWAYS_PRIMARY|DISALLOW_TO_FAILOVER"
        - name: PGPOOL_PARAMS_BACKEND_FLAG1
          value: "DISALLOW_TO_FAILOVER"
        - name: PGPOOL_PARAMS_BACKEND_HOSTNAME1
          value: "hot-postgres-standby"
        - name: PGPOOL_PARAMS_BACKEND_PORT1
          value: "5432"
        - name: PGPOOL_PARAMS_BACKEND_WEIGHT1
          value: "1"
        - name: PGPOOL_PARAMS_BACKEND_FLAG1
          value: "DISALLOW_TO_FAILOVER"
        - name: PGPOOL_PARAMS_FAILOVER_ON_BACKEND_ERROR
          value: "off"
        - name: POSTGRES_USERNAME
          value: irembo_user
        - name: POSTGRES_PASSWORD
          value: password
---
apiVersion: v1
kind: Service
metadata:
  name: pgpool
  namespace: data
spec:
  selector:
    app: pgpool
  ports:
  - name: pgpool-port
    protocol: TCP
    port: 9999
    targetPort: 9999
[root@control-plane ~]#

````

# Port forward your pgpool to local machine for access


```
kubectl port-forward svc/pgpool -n data 5000:9999
```


## Data recovery in steps 
- connect to psql like so  ``` psql -U irembo_user -h localhost -p 5000 ```
- recover the definitions.sql first ``` psql -U irembo_user -h localhost -p 5000 < definitions.sql ```
- create each dbname that you're going to recover ie: 'irembo , crvs , etc..' 

# Recover db's with pg_recover

- ``` pg_restore --dbname=irembo --verbose irembo.tar ```



Read the errors as they appear . 
