# Installing Deis for Production on Azure

In [Installing Deis on Azure](https://github.com/jmspring/acs-deployments/blob/master/deis_on_azure/deis_on_azure.md)
a basic Deis Workflow was installed.  By default, Deis Workflow uses Minio for storage within the cluster and does not
persist if the database was to go down.  Also, by default, user registration is open to one and all.

In order to get around this, you can deploy your Deis Workflow with a specified set of values.  The `values.yaml`
file is present within the Workflow tarball.  The process for customizing a deployment is as follows:

  - Determine desired configuration.  
  - Retrieve the `values.yaml` file.
  - Edit required settings.
  - Deploy Workflow with the custom values.

## Determine desired configuration

For this deployment, the desired requriements are as follows:

  - Persistent storage using Azure Blob Store
  - Lock down user registration to `admin_only`

In order to configure persistent storage for Azure Blob Store, a storage account is necessary and the key is 
needed.  For the storage account, we will leverage the values used in 
[Installing Deis on Azure](https://github.com/jmspring/acs-deployments/blob/master/deis_on_azure/deis_on_azure.md) 
as well as specify some Storage specific values.

Values taken from the earlier deployment:

  - **resource group**: deisonk8srg
  - **location**: westus

Values specific to the storage account:

  - **name**: deisconfig
  - **kind**: Storage
  - **sku**: Standard_LRS

Creating the Storage Account:

```bash
jims@dockeropolis:~$ az storage account create \
> --sku="Standard_LRS" \ 
> --resource-group="deisonk8srg" \
> --kind="Storage" 
> --location="westus" \
> --name="deisconfig"
{
  "accessTier": null,
  "creationTime": "2016-12-07T20:06:46.844902+00:00",
  "customDomain": null,
  "encryption": null,
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Storage/storageAccounts/deisconfig",
  "kind": "Storage",
  "lastGeoFailoverTime": null,
  "location": "westus",
  "name": "deisconfig",
  "primaryEndpoints": {
    "blob": "https://deisconfig.blob.core.windows.net/",
    "file": "https://deisconfig.file.core.windows.net/",
    "queue": "https://deisconfig.queue.core.windows.net/",
    "table": "https://deisconfig.table.core.windows.net/"
  },
  "primaryLocation": "westus",
  "provisioningState": "Succeeded",
  "resourceGroup": "deisonk8srg",
  "secondaryEndpoints": null,
  "secondaryLocation": null,
  "sku": {
    "name": "Standard_LRS",
    "tier": "Standard"
  },
  "statusOfPrimary": "Available",
  "statusOfSecondary": null,
  "tags": {},
  "type": "Microsoft.Storage/storageAccounts"
}
```

Next we will need to retrieve the keys for access to the Storage Account:

```bash
jims@dockeropolis:~$ az storage account keys list \
> --name="deisconfig" \
> --resource-group="deisonk8srg"
{
  "keys": [
    {
      "keyName": "key1",
      "permissions": "FULL",
      "value": "LAVhsViMiy7IlRAujOmxjgHuG5dH1O6xTS8RYJmmgFKTLfm+kb/M1nyJj4GE3zUf/gJ4rRfgtyByqNcgtsGew=="
    },
    {
      "keyName": "key2",
      "permissions": "FULL",
      "value": "+IRSnU4cOVGaJyFFuPBy+nLrfcnGcOrEtickFPOF8kV3QoZyJ9CR7bFx9ei0noF12+e4r9TVQ6t9IkSojVJw=="
    }
  ]
}

## Retrieve the `values.yaml` file

This step assumes the `deis/workflow` repository has added into `helm`.  One needs to retrieve the 
`deis/workflow` tarball and retrieve the `values.yaml` file from the expanded tarball. The process:

```bash
jims@dockeropolis:~$ helm fetch deis/workflow --untar
jims@dockeropolis:~$ ls
config
workflow
jims@dockeropolis:~$ cp workflow/values.yaml  ./config/
```

### Edit the required settings in `values.yaml`

In an editor of choice, open `values.yaml`.  As mentioned, the desired configuration is to persist
internal Workflow state to Azure Storage and to configure registration requirements (locking things 
down).  

To configure the use of Azure Storage, within `values.yaml`, the default values are as follows:

```bash
global:
  # Set the storage backend
  #
  # Valid values are:
  # - s3: Store persistent data in AWS S3 (configure in S3 section)
  # - azure: Store persistent data in Azure's object storage
  # - gcs: Store persistent data in Google Cloud Storage
  # - minio: Store persistent data on in-cluster Minio server
  storage: minio
...
```

The first step is to change from `minio` to `azure`.  

```bash
global:
  # Set the storage backend
  #
  # Valid values are:
  # - s3: Store persistent data in AWS S3 (configure in S3 section)
  # - azure: Store persistent data in Azure's object storage
  # - gcs: Store persistent data in Google Cloud Storage
  # - minio: Store persistent data on in-cluster Minio server
  storage: azure
...
```

Once this is done, the Azure specific values need to be configured.  The values needed are
the Storage Account **name** and the Storage Account **access key** from above:

  - Name:  `deisconfig`
  - Key:   `LAVhsViMiy7IlRAujOmxjgHuG5dH1O6xTS8RYJmmgFKTLfm+kb/M1nyJj4GE3zUf/gJ4rRfgtyByqNcgtsGew==`

Additionally, names for three containers will be needed for `registry`, `database`, and `builder`.  
For this, the following names will be used:

  - Registery container:  `regcont`
  - Database container:   `dbcont`
  - Builder container:    `bldcont`

The initial config within `values.yaml` looks as follows:

```bash
azure:
  accountname: "YOUR ACCOUNT NAME"
  accountkey: "YOUR ACCOUNT KEY"
  registry_container: "your-registry-container-name"
  database_container: "your-database-container-name"
  builder_container: "your-builder-container-name"
```

Modify the above using the appropriate values as follows:

```bash
azure:
  accountname: "deisconfig"
  accountkey: "LAVhsViMiy7IlRAujOmxjgHuG5dH1O6xTS8RYJmmgFKTLfm+kb/M1nyJj4GE3zUf/gJ4rRfgtyByqNcgtsGew=="
  registry_container: "regcont"
  database_container: "dbcont"
  builder_container: "bldcont"
```

In order to lock down registration, the configuration of the `controller` `registration mode` needs to 
be modified from `enabled` mode to `admin_only` as follows:

```bash
controller:
  app_pull_policy: "IfNotPresent"
  # Possible values are:
  # enabled - allows for open registration
  # disabled - turns off open registration
  # admin_only - allows for registration by an admin only.
  registration_mode: "enabled"
```

Change to:

```bash
controller:
  app_pull_policy: "IfNotPresent"
  # Possible values are:
  # enabled - allows for open registration
  # disabled - turns off open registration
  # admin_only - allows for registration by an admin only.
  registration_mode: "admin_only"
```

At this point, the changes to `values.yaml` are complete.

## Deploy Workflow with the custom values

In order to deploy workflow with custome values that were just editted, the command is as
follows:

```bash
jims@dockeropolis:~$ helm install deis/workflow --namespace deis -f ./config/values.yaml
Fetched deis/workflow to workflow-v2.9.0.tgz
NAME: knotted-dragon
LAST DEPLOYED: Wed Dec  7 12:15:18 2016
NAMESPACE: deis
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME         TYPE      DATA      AGE
minio-user   Opaque    2         4s
deis-router-dhparam   Opaque    1         4s
objectstorage-keyfile   Opaque    5         4s

==> v1/ConfigMap
NAME                 DATA      AGE
slugbuilder-config   2         4s
dockerbuilder-config   2         4s
slugrunner-config   1         4s

==> v1/ServiceAccount
NAME            SECRETS   AGE
deis-database   1         4s
deis-controller   1         4s
deis-workflow-manager   1         4s
deis-registry   1         4s
deis-monitor-telegraf   1         4s
deis-logger-fluentd   1         4s
deis-builder   1         4s
deis-router   1         4s
deis-logger   1         4s
deis-nsqd   1         4s

==> v1/Service
NAME        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
deis-nsqd   10.0.28.62   <none>        4151/TCP,4150/TCP   4s
deis-builder   10.0.71.84   <none>    2222/TCP   4s
deis-router   10.0.176.240   <pending>   80/TCP,443/TCP,2222/TCP,9090/TCP   3s
deis-database   10.0.164.223   <none>    5432/TCP   3s
deis-logger-redis   10.0.250.167   <none>    6379/TCP   3s
deis-logger   10.0.10.91   <none>    80/TCP    3s
deis-workflow-manager   10.0.93.80   <none>    80/TCP    3s
deis-registry   10.0.15.106   <none>    80/TCP    3s
deis-controller   10.0.67.21   <none>    80/TCP    3s
deis-monitor-grafana   10.0.162.195   <none>    80/TCP    3s
deis-monitor-influxapi   10.0.84.216   <none>    80/TCP    3s
deis-monitor-influxui   10.0.188.113   <none>    80/TCP    3s

==> extensions/Deployment
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deis-router   1         1         1            0           3s
deis-workflow-manager   1         1         1         0         3s
deis-nsqd   1         1         1         0         3s
deis-database   1         1         1         0         3s
deis-builder   1         1         1         0         3s
deis-logger-redis   1         1         1         0         3s
deis-registry   1         1         1         0         3s
deis-monitor-grafana   1         1         1         0         3s
deis-logger   1         0         0         0         3s
deis-controller   1         0         0         0         3s
deis-monitor-influxdb   1         0         0         0         2s

==> extensions/DaemonSet
NAME                  DESIRED   CURRENT   NODE-SELECTOR   AGE
deis-logger-fluentd   7         7         <none>          2s
deis-monitor-telegraf   7         6         <none>    2s
deis-registry-proxy   0         0         <none>    1s
```

This looks just like the prior deployment.  The only difference is that the configuration
has been customized and because of the reliance on the external Azure Storage, bring up of
the Deis Workflow (specifically, `database`, `registry`, and `controller` will take a bit
longer.

```bash
jims@dockeropolis:~$ kubectl --namespace=deis get pods
NAME                                     READY     STATUS              RESTARTS   AGE
deis-builder-2379916125-5s99i            0/1       Running             0          19s
deis-controller-183254951-3fnc8          0/1       Running             0          18s
deis-database-473246318-w42z2            0/1       Running             0          19s
deis-logger-4136341797-tdaju             0/1       Running             1          18s
deis-logger-fluentd-01e3l                1/1       Running             0          18s
deis-logger-fluentd-jcnmw                1/1       Running             0          18s
deis-logger-fluentd-jhxox                1/1       Running             0          18s
deis-logger-fluentd-p5wwc                1/1       Running             0          18s
deis-logger-fluentd-s4l6y                1/1       Running             0          18s
deis-logger-fluentd-ss0zw                1/1       Running             0          18s
deis-logger-fluentd-xioid                1/1       Running             0          18s
deis-logger-redis-2548468243-mqnax       1/1       Running             0          19s
deis-monitor-grafana-323247549-qmq5h     1/1       Running             0          18s
deis-monitor-influxdb-2577745094-9o7d8   1/1       Running             0          17s
deis-monitor-telegraf-3pyku              1/1       Running             0          17s
deis-monitor-telegraf-ed90j              1/1       Running             0          17s
deis-monitor-telegraf-iyiw9              1/1       Running             0          17s
deis-monitor-telegraf-kduhg              1/1       Running             0          17s
deis-monitor-telegraf-p4yom              1/1       Running             0          18s
deis-monitor-telegraf-uv5ez              1/1       Running             0          17s
deis-monitor-telegraf-w7sp0              1/1       Running             0          17s
deis-nsqd-3430976322-g9ua9               0/1       Running             0          19s
deis-registry-2661201211-kr3dd           0/1       ContainerCreating   0          18s
deis-registry-proxy-2eqns                1/1       Running             0          17s
deis-registry-proxy-6rrdq                1/1       Running             0          16s
deis-registry-proxy-g0d3u                1/1       Running             0          16s
deis-registry-proxy-pk4p4                1/1       Running             0          17s
deis-registry-proxy-r36xj                1/1       Running             0          16s
deis-registry-proxy-v5kdh                1/1       Running             0          16s
deis-registry-proxy-yud40                1/1       Running             0          17s
deis-router-1741606082-9uuf4             1/1       Running             0          19s
deis-workflow-manager-2369615478-jvyui   1/1       Running             0          19s
```

The `deis-database` pod needs to be up and running for `builder` and `controller` to come
up.  Inquiring on the state of the database, resembles as follows:

```bash
kubectl logs -f deis-database-473246318-w42z2 --namespace=deis
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
creating template1 database in /var/lib/postgresql/data/base/1 ... ok
initializing pg_authid ... ok
initializing dependencies ... ok
creating system views ... ok
loading system objects' descriptions ... ok
creating collations ... ok
creating conversions ... ok
creating dictionaries ... ok
setting privileges on built-in objects ... ok
creating information schema ... ok
loading PL/pgSQL server-side language ... ok
vacuuming database template1 ... ok
copying template1 to template0 ... ok
copying template1 to postgres ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.
syncing data to disk ... ok

Success. You can now start the database server using:

    postgres -D /var/lib/postgresql/data
or
    pg_ctl -D /var/lib/postgresql/data -l logfile start

waiting for server to start....LOG:  database system was shut down at 2016-12-07 20:15:28 UTC
LOG:  MultiXact member wraparound protections are now enabled
LOG:  autovacuum launcher started
LOG:  database system is ready to accept connections
 done
server started
CREATE DATABASE

CREATE ROLE


/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/001_setup_envdir.sh

/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/002_create_bucket.sh

/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/003_restore_from_backup.sh
Rebooting postgres to enable archive mode
waiting for server to shut down....LOG:  received smart shutdown request
LOG:  autovacuum launcher shutting down
LOG:  shutting down
LOG:  database system is shut down
 done
server stopped
waiting for server to start....LOG:  database system was shut down at 2016-12-07 20:15:34 UTC
LOG:  MultiXact member wraparound protections are now enabled
LOG:  autovacuum launcher started
LOG:  database system is ready to accept connections
 done
server started
Performing an initial backup...
wal_e.main   INFO     MSG: starting WAL-E
        DETAIL: The subcommand is "backup-push".
        STRUCTURED: time=2016-12-07T20:15:37.256543-00 pid=111
wal_e.worker.upload INFO     MSG: begin archiving a file
        DETAIL: Uploading "pg_xlog/000000010000000000000001" to "wabs://deisdb/wal_005/000000010000000000000001.lzo".
        STRUCTURED: time=2016-12-07T20:15:38.169142-00 pid=126 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000001.lzo prefix= seg=000000010000000000000001 state=begin
wal_e.worker.upload INFO     MSG: completed archiving to a file
        DETAIL: Archiving to "wabs://deisdb/wal_005/000000010000000000000001.lzo" complete at 00KiB/s.
        STRUCTURED: time=2016-12-07T20:15:38.557512-00 pid=126 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000001.lzo prefix= rate=00 seg=000000010000000000000001 state=complete
wal_e.operator.backup INFO     MSG: start upload postgres version metadata
        DETAIL: Uploading to wabs://deisdb/basebackups_005/base_000000010000000000000002_00000040/extended_version.txt.
        STRUCTURED: time=2016-12-07T20:15:38.631561-00 pid=111
wal_e.operator.backup INFO     MSG: postgres version metadata upload complete
        STRUCTURED: time=2016-12-07T20:15:38.805977-00 pid=111
wal_e.worker.upload INFO     MSG: beginning volume compression
        DETAIL: Building volume 0.
        STRUCTURED: time=2016-12-07T20:15:38.847086-00 pid=111
wal_e.worker.upload INFO     MSG: begin uploading a base backup volume
        DETAIL: Uploading to "wabs://deisdb/basebackups_005/base_000000010000000000000002_00000040/tar_partitions/part_00000000.tar.lzo".
        STRUCTURED: time=2016-12-07T20:15:39.256351-00 pid=111
wal_e.worker.upload INFO     MSG: finish uploading a base backup volume
        DETAIL: Uploading to "wabs://deisdb/basebackups_005/base_000000010000000000000002_00000040/tar_partitions/part_00000000.tar.lzo" complete at 00KiB/s. 
        STRUCTURED: time=2016-12-07T20:15:39.732728-00 pid=111
wal_e.worker.upload INFO     MSG: begin archiving a file
        DETAIL: Uploading "pg_xlog/000000010000000000000002" to "wabs://deisdb/wal_005/000000010000000000000002.lzo".
        STRUCTURED: time=2016-12-07T20:15:40.156166-00 pid=140 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000002.lzo prefix= seg=000000010000000000000002 state=begin
wal_e.worker.upload INFO     MSG: completed archiving to a file
        DETAIL: Archiving to "wabs://deisdb/wal_005/000000010000000000000002.lzo" complete at 00KiB/s.
        STRUCTURED: time=2016-12-07T20:15:40.432391-00 pid=140 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000002.lzo prefix= rate=00 seg=000000010000000000000002 state=complete
wal_e.worker.upload INFO     MSG: begin archiving a file
        DETAIL: Uploading "pg_xlog/000000010000000000000002.00000028.backup" to "wabs://deisdb/wal_005/000000010000000000000002.00000028.backup.lzo".
        STRUCTURED: time=2016-12-07T20:15:40.826042-00 pid=148 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000002.00000028.backup.lzo prefix= seg=000000010000000000000002.00000028.backup state=begin
wal_e.worker.upload INFO     MSG: completed archiving to a file
        DETAIL: Archiving to "wabs://deisdb/wal_005/000000010000000000000002.00000028.backup.lzo" complete at 00KiB/s.
        STRUCTURED: time=2016-12-07T20:15:41.154276-00 pid=148 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000002.00000028.backup.lzo prefix= rate=00 seg=000000010000000000000002.00000028.backup state=complete
NOTICE:  pg_stop_backup complete, all required WAL segments have been archived

/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/004_run_backups.sh

waiting for server to shut down...LOG:  received fast shutdown request
LOG:  aborting any active transactions
.LOG:  autovacuum launcher shutting down
LOG:  shutting down
LOG:  database system is shut down
 done
server stopped

PostgreSQL init process complete; ready for start up.

LOG:  database system was shut down at 2016-12-07 20:15:42 UTC
LOG:  MultiXact member wraparound protections are now enabled
LOG:  autovacuum launcher started
LOG:  database system is ready to accept connections
wal_e.worker.upload INFO     MSG: begin archiving a file
        DETAIL: Uploading "pg_xlog/000000010000000000000003" to "wabs://deisdb/wal_005/000000010000000000000003.lzo".
        STRUCTURED: time=2016-12-07T20:16:43.899192-00 pid=228 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000003.lzo prefix= seg=000000010000000000000003 state=begin
wal_e.worker.upload INFO     MSG: completed archiving to a file
        DETAIL: Archiving to "wabs://deisdb/wal_005/000000010000000000000003.lzo" complete at 00KiB/s.
        STRUCTURED: time=2016-12-07T20:16:44.182688-00 pid=228 action=push-wal key=wabs://deisdb/wal_005/000000010000000000000003.lzo prefix= rate=00 seg=000000010000000000000003 state=complete
``` 

After a bit, all pods will be running:

```bash
jims@dockeropolis:~$ kubectl --namespace=deis get pods
NAME                                     READY     STATUS    RESTARTS   AGE
deis-builder-2379916125-5s99i            1/1       Running   0          2m
deis-controller-183254951-3fnc8          1/1       Running   1          2m
deis-database-473246318-w42z2            1/1       Running   0          2m
deis-logger-4136341797-tdaju             1/1       Running   2          2m
deis-logger-fluentd-01e3l                1/1       Running   0          2m
deis-logger-fluentd-jcnmw                1/1       Running   0          2m
deis-logger-fluentd-jhxox                1/1       Running   0          2m
deis-logger-fluentd-p5wwc                1/1       Running   0          2m
deis-logger-fluentd-s4l6y                1/1       Running   0          2m
deis-logger-fluentd-ss0zw                1/1       Running   0          2m
deis-logger-fluentd-xioid                1/1       Running   0          2m
deis-logger-redis-2548468243-mqnax       1/1       Running   0          2m
deis-monitor-grafana-323247549-qmq5h     1/1       Running   0          2m
deis-monitor-influxdb-2577745094-9o7d8   1/1       Running   0          2m
deis-monitor-telegraf-3pyku              1/1       Running   1          2m
deis-monitor-telegraf-ed90j              1/1       Running   1          2m
deis-monitor-telegraf-iyiw9              1/1       Running   1          2m
deis-monitor-telegraf-kduhg              1/1       Running   1          2m
deis-monitor-telegraf-p4yom              1/1       Running   1          2m
deis-monitor-telegraf-uv5ez              1/1       Running   1          2m
deis-monitor-telegraf-w7sp0              1/1       Running   1          2m
deis-nsqd-3430976322-g9ua9               1/1       Running   0          2m
deis-registry-2661201211-kr3dd           1/1       Running   0          2m
deis-registry-proxy-2eqns                1/1       Running   0          2m
deis-registry-proxy-6rrdq                1/1       Running   0          2m
deis-registry-proxy-g0d3u                1/1       Running   0          2m
deis-registry-proxy-pk4p4                1/1       Running   0          2m
deis-registry-proxy-r36xj                1/1       Running   0          2m
deis-registry-proxy-v5kdh                1/1       Running   0          2m
deis-registry-proxy-yud40                1/1       Running   0          2m
deis-router-1741606082-9uuf4             1/1       Running   0          2m
deis-workflow-manager-2369615478-jvyui   1/1       Running   0          2m
```