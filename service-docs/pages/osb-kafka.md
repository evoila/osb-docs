# OSB-Kafka
- [OSB-Kafka](#osb-kafka)
  - [Overview TODO](#overview-todo)
    - [Key Features](#key-features)
    - [Software used by OSB-Kafka](#software-used-by-osb-kafka)
    - [Cluster TODO (nutzen wir Redis Cluster für Monitoring?)](#cluster-todo-nutzen-wir-redis-cluster-für-monitoring)
  - [Requirements](#requirements)
  - [How to](#how-to)
    - [Create a Service Instance](#create-a-service-instance)
    - [Update a Service Instance](#update-a-service-instance)
    - [Create a Service Binding](#create-a-service-binding)
    - [Acquiring Service Instance Parameters](#acquiring-service-instance-parameters)
    - [Backup TODO](#backup-todo)
    - [Change SSL Certificates TODO](#change-ssl-certificates-todo)
  - [Settings TODO](#settings-todo)
    - [Service Instance Settings Schema TODO](#service-instance-settings-schema-todo)
      - [logging object](#logging-object)
      - [config object](#config-object)
      - [security object](#security-object)
      - [users object](#users-object)
    - [Service Instance Zookeper Schema](#service-instance-zookeper-schema)
    - [Service Binding Settings Schema](#service-binding-settings-schema)
      - [topic_acls object](#topic_acls-object)
      - [group_acls](#group_acls)
      - [cluster_acls object](#cluster_acls-object)
  - [FAQ](#faq)
    - [OSB-Redis crashed](#osb-redis-crashed)
    - [A Redis master/replica instance crashed](#a-redis-masterreplica-instance-crashed)
    - [The size of the backup was bigger than expected (and failed) and now all of my storage space is occupied TODO](#the-size-of-the-backup-was-bigger-than-expected-and-failed-and-now-all-of-my-storage-space-is-occupied-todo)
  - [Appendix](#appendix)
    - [JSON-Schema](#json-schema)

---

## Overview TODO

[Apache Kafka](https://kafka.apache.org/) is one of the most popular event streaming platforms that offers high-performance data pipelines, streaming analytics, data integration and mission-critical applications. Event streaming allows for publishing (writing) and subscribing (reading) streams of events in real-time.

### Key Features

Some of Apache Kafkas key features are:

- **High throughput**: Messages can be delivered at network limited throughput and can be as low as 2ms.
- **Scalability**: Apache Kafka can be scaled up by increasing the number of brokers in order to maintain load balance. Since Kafka brokers are stateless, ZooKeeper is used for maintaining their cluster state. Kafka broker leader election is also done by ZooKeeper.
- **High-availability**: Multiple Kafka brokers replicate other ones to make data fault-tolerant and highly-available.
- **APIs**: Kafka exposes all its functionality over a language independent protocol which has clients in many programming languages. Kafka includes five core APIs:
  - Producer API
  - Consumer API
  - Streams API
  - Connect API
  - Admin API

  For more information, see the [documentation](https://kafka.apache.org/documentation/#api).
- **Client libraries**: There are client libraries available as independent open-source projects for many languages, for example C/C++, Python, Go, Node.js and PHP. A comprehensive list can be found [here](https://cwiki.apache.org/confluence/display/KAFKA/Clients). However, only the Java clients are maintained as part of the main Kafka project.

The OSB-Kafka offers different service plans which vary in allocated memory, cpu, disc-size and number of vms created for PostgreSQL.

### Software used by OSB-Kafka
- **Kafka**: 2.12?

### Cluster TODO (nutzen wir Redis Cluster für Monitoring?)

Using PostgreSQL, it is possible to transfer [Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html)
synchronously or asynchronously to standby-nodes via [Streaming Replication (SR)](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION). <br>
Additionally, the Software "Patroni" is used with a HAProxy inside the OSB, so that an automatic switch to another node occurs
in case of a failure of the primary-node. Inside PostgreSQL, there is no load-balancing provided.

The following image shows how PostgreSQL nodes are managed:
![Postgres-Cluster](assets/osb-postgres.png)

An HAProxy conists of at least 2 PostgeSQL nodes that are connected via Streaming Replication. Patroni exists on every PostgreSQL node for automatic failover. A central IP or DNS entry is returned by HAProxy. The health check is done by Patroni: In case of failure the primary node will be switched automatically and HAProxy will be set to this one. In order for Patroni to manage the clusters, a central Zookeeper cluster with 3 or 5 instances will be used for all PostgreSQL clusters.

## Requirements
- [Cloud Foundry CLI](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)

## How to
### Create a Service Instance

A service instance can be created manually via the CLI-Command
```
cf create-service SERVICE PLAN SERVICE_INSTANCE [-b BROKER] [-c PARAMETERS_AS_JSON] [-t TAGS]
```

- **SERVICE** will be the name of the service broker which is likely going to be **osb-kafka**.
- **PLAN** is a plan offered by the service.
- **SERVICE_INSTANCE** the name of the service instance, can be chosen freely.
- **PARAMETERS_AS_JSON** contains the settings shown in [Settings](#settings).

For more information see [Cloud Foundry CLI Reference Guide](https://cli.cloudfoundry.org/en-US/v6/create-service.html).

TODO (?Schema für create)
Aternatively, if there is a dashboard set up (like the Stratos Dashboard for example), it can be used to create a service instance.

### Update a Service Instance

A service instance (and therefore its settings) can be updated manually via the CLI-Command
```
cf update-service SERVICE_INSTANCE [-p NEW_PLAN] [-c PARAMETERS_AS_JSON] [-t TAGS] [--upgrade]
```

- **SERVICE_INSTANCE** is be the name of the previously created service instance.
- **PARAMETERS_AS_JSON** contains the settings shown in [Settings](#settings)

For more information see [Cloud Foundry CLI Reference Guide](https://cli.cloudfoundry.org/en-US/v6/update-service.html).

Aternatively, if there is a dashboard set up (like the Stratos Dashboard for example), it can be used to update a service instance.

Keep in mind that **previous values will be overwritten**. In order to see the existing parameters you can use a dashboard or acquire the parameters via cli (see [Acquiring Service Instance Parameters](#acquiring-service-instance-parameters)).

### Create a Service Binding

A binding can be created manually via the CLI-Command 
```
cf bind-service APP_NAME SERVICE_INSTANCE [-c PARAMETERS_AS_JSON] [--binding-name BINDING_NAME]
```

- **APP_NAME** ist the name of the previously created app that gets the binding injected.
- **SERVICE_INSTANCE** is be the name of the previously created service instance.
- **PARAMETERS_AS_JSON** contains the settings shown in [Settings](#service-binding-settings-schema).


For more information see [Cloud Foundry CLI Reference Guide](https://cli.cloudfoundry.org/en-US/v6/bind-service.html).
After creating a binding, the app has to be restarted for the changes to take effect.

TODO
Aternatively, if there is a dashboard set up (like the Stratos Dashboard for example), it can be used to create a service binding.

### Acquiring Service Instance Parameters

The current parameters (and therefore settings) can be retrieved via cli:

1. ```cf service --guid **SERVICE_INSTANCE**```
2. ```cf curl v3/service_instances/**SERVICE_INSTANCE_ID**/parameters```
3. A JSON with the parameters will be returned.

- **SERVICE_INSTANCE** is be the name of the previously created service instance.
- **SERVICE_INSTANCE_ID** is the guid of the service instance which is acquired in step 1. 


### Backup TODO

Wahrscheinlich kein redis backup? Was ist dann mit rdbchecksum und rdbcompression? Oder nur genutzt für diskless replication?

When creating an instance, the disk-space should be calculated. It is important to keep in mind that there is enough space for files, backup and WAL (Write Ahead Log) segments. The backup size can vary, depending on data type and indices:

- If there are many big files of type TEXT / JSONB / JSON / BYTEA, the backup size can be 1.5 times bigger than the database itself.
- If there are many indices the backup size can shrink to 0.25 the current size.
- The size of a plan can be scaled up afterwards but this will copy the disk. In order to avoid subsequent changes a fitting plan should be chosen according to the following rule: `Files * ( 1 + Backup Factor ) + max_wal_size `. Thereby the required space can go up to 2.5 times the size plus ~10GB WAL-files.

Setting up a backup can be done the dashboard-url of the service instance (which can be retrieved by the cli command **cf service SERVICE_INSTANCE**). For more information, see the Backup Docs.

### Change SSL Certificates TODO

Wahrscheinlich nicht in redis vorhanden?

The certificates expire after 365 day. If a certificate is about to expire, contact the operator of the Service Broker to renew the certificates.

If Bosh DNS is used, the certificates are stored in Credhub and can be renewed there. If the root CA is still valid, the certificate can simply be deleted and the new certificate can be used via `bosh manifest`and `bosh deploy`. If the root CA expires, it is necessary to concatenate old and new certificates, for example via [https://github.com/pivotal/credhub-release/blob/main/docs/ca-rotation.md](https://github.com/pivotal/credhub-release/blob/main/docs/ca-rotation.md).</br>
If the IP variant is used and the root CA still valid, it is sufficient to use `bosh recreate`. For changing the root CA, it also has to be concatenated and multiple deploys have to be made.

## Settings TODO
This section covers different settings that can be made for the OSB-Redis, their default values and how they can be changed.

Settings can be sent as parameters of a create/update (?schema noch ergänizen für create?) request of a service instance via CLI.

TThe CLI command will look like this:
```
cf cs SERVIE PLAN SERVICE_INSTANCE_ [-c PARAMETERS_AS_JSON]
```
or
```
cf update-service SERVICE_INSTANCE [-c PARAMETERS_AS_JSON]
```
- **SERVICE** will be the name of the service broker which is likely going to be **osb-kafka**.
- **PLAN** is the plan that is going to be used for the service instance.
- **SERVICE_INSTANCE** is the name of the service instance.
- **PARAMETERS_AS_JSON** are the settings which are sent in json format.

For example, a cli command for creating a service instance could look like this:
```
cf cs osb-kafka s kafka-test -c '{"redis":{"config":{"tcp_keepalive":70}, "store":{"rdbcompression":"no"}}}'
```
An extended example of the parameters for a create/update request for a service instance is shown below:
```json
{
    "redis": {
      "config": {
        "tcp_keepalive": 42,
        "timeout": 65
      },
      "limits": {
        "fd": 65000
      },
      "store": {
        "rdbchecksum": "yes",
        "rdbcompression": "no",
        "stop_writes_on_bgsave_error": "yes"
      }
    }
}
```

### Service Instance Settings Schema TODO

The following settings are defined in the schema in TODO(create hat noch kein schema) service_plan.schemas.service_instance.**create**.parameters.properties.redis.properties and service_plan.schemas.service_instance.**update**.parameters.properties.redis.properties (?überprüfen ob korrekt)

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| logging | [logging object](#logging-object) | - | Contains properties for the log rotation?. |
| config | [config object](#config-object) | - | Contains general settings. |
| security | [security object](#security-object) | -  | Contains security settings. |
| user | array of [users objects](#users-object) | - | Contains the users for Kafka. |

#### logging object

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| log_level | string | "INFO" | Sets the level of kafka logs. Valid values are "ALL", "DEBUG", "INFO", "WARN", "ERROR", "FATAL", "OFF" and "TRACE" (warum kein enum im schema?). If a logging object is given, this property is **required**. |
| max_file_size | string | "10MB" | Sets the maximum Log4j file size (KB, MB, GB). If a logging object is given, this property is **required**. |
| max_backup_index | number | 9 | Maximum number of Log4j backup log files. If a logging object is given, this property is **required**. |

#### config object

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| delete_topic | boolean | 0 | Sets, whether topics should be deleted (genug infos? deleted WANN?). If a config object is given, this property is **required**. |
| log_retention_check_interval_ms | number | 300000 | The log retention check interval. If a config object is given, this property is **required**. |
| log_retention_hours | number | 168 | Duration of the log retention in hours. If a config object is given, this property is **required**. |
| log_segment_bytes | number | 1073741824 | The size of log segments in bytes. If a config object is given, this property is **required**. |
| max_poll_interval | number | 5000 | The maximum polling interval in ms. If a config object is given, this property is **required**. |
| num_io_threads | number | 8 | Number of threads doing disk I/O. If a config object is given, this property is **required**. |
| num_network_threads | number | 4 | Number of Threads handling network requests. If a config object is given, this property is **required**. |
| num_partitions | number | 1 | Number of log partitions. If a config object is given, this property is **required**. |
| num_recovery_threads_per_data_dir | number | 1 | Number of recovery threads. If a config object is given, this property is **required**. |
| offsets_topic_replication_factor | number | 3 | Offsets topic replication factor. If a config object is given, this property is **required**. |
| socket_receive_buffer_bytes | number | 102400 | Receive buffer in bytes. If a config object is given, this property is **required**. |
| socket_request_max_bytes | number | 104857600 | Maximum request size accepted in bytes. If a config object is given, this property is **required**. |
| socket_send_buffer_bytes | number | 102400 | Send buffer in bytes. If a config object is given, this property is **required**. |

#### security object

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| setup_secure_client_connection | boolean | 0 | Secure client connection. If a security object is given, this property is **required**. |

#### users object

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| username | string | - | The username (frei wählbar?). Valid symbols have to be in the pattern ^[A-Za-z0-9_-]+$. If a users object is given, this property is **required**. |
| password | string | - | If a users object is given, this property is **required**. |

### Service Instance Zookeper Schema

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| autopurge_purge_interval | number | 24(stunden?) | Autopurge interval. If a Zookeper object is given, this property is **required**. |
| autopurge_snap_retain_count | number | 3 | Autopurge snap(shot?) retain count. If a Zookeper object is given, this property is **required**. |
| cnx_timeout | number | 5(stunden?) | CNX timeout. If a Zookeper object is given, this property is **required**. |
| election_algorim | number | 3 | Election algorithm (to avoid confusion: there is a typo in the property name)(wofür steht welche nummer?). If a Zookeper object is given, this property is **required**. |
| force_sync | string | "yes" | Force synchronization. Valid values are "yes" and "no" (warum kein enum im schema?). If a Zookeper object is given, this property is **required**. |
| global_outstanding_limit | number | 1000 | Global outstanding limit (weiß der nutzer was damit gemeint ist? ich nicht). If a Zookeper object is given, this property is **required**. |
| init_limit | number | 5 | Connection retries. If a Zookeper object is given, this property is **required**. |
| leader_serves | string | "yes" | Client connections to leader. Valid values are "yes" and "no". If a Zookeper object is given, this property is **required**. |
| max_client_connections | number | 60 | Maximum number of client connections. If a Zookeper object is given, this property is **required**. |
| max_session_timeout | number | 40000 | Maximum session timeout in ms. If a Zookeper object is given, this property is **required**. |
| min_session_timeout | number | 4000 | Minimum session timeout in ms. If a Zookeper object is given, this property is **required**. |
| pre_allocation_size | number | 65536 | Pre-allocation size for transaction log blocks (KB). If a Zookeper object is given, this property is **required**. |
| snap_count | number | 100000 | Snap(shot?) count. If a Zookeper object is given, this property is **required**. |
| sync_enabled | boolean | true | Synchronization enabled. If a Zookeper object is given, this property is **required**. |
| sync_limit | number | 2 | Synchronization limit. If a Zookeper object is given, this property is **required**. |
| tick_time | number | 2000 | Single tick time (in ms?). If a Zookeper object is given, this property is **required**. |
| warning_threshold_ms | number | 1000 | Warning threshold in ms. If a Zookeper object is given, this property is **required**.  | 

### Service Binding Settings Schema

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| service_key_name | string | - | The name of the service binding. This parameter is **required**. |
| topic_acls | array of [topic_acls objects](#topic_acls-object) | - | ? |
| group_acls | array of [group_acls objects](#group_acls-object) | - | ? |
| cluster_acls | array of [cluster_acls objects](#cluster_acls-object) | - | ? |

#### topic_acls object

blabla?

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| topic | string | * | ?. If a topic_acls object is given, this parameter is **required**. |
| rights | array of string | - | Sets the rights of the topic (?). Valid values are "All", "Alter", "AlterConfigs", "Create", "Delete", "Describe", "DescribeConfigs", "Read" and "Write". If a topic_acls object is given, this parameter is **required**. |

#### group_acls 

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| group | string | * | name? of group?. If a group_acls object is given, this parameter is **required**. |
| rights | array of string | "All" | Sets the rights of the group. Valid values are "All", "Delete", "Describe" and "Read". If a group_acls object is given, this parameter is **required**. |

#### cluster_acls object

Warum ist cluster_acls als array of objects definiert, wenn "maxItems: 1"? Macht das array da überhaupt Sinn?

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| rights | array of string | "All" | Sets the rights of the cluster?. Valid values are "All", "Alter", "AlterConfigs", "ClusterAction", "Create", "Describe", "DescribeConfigs", "IdempotentWrite". If a cluster_acls object is given, this parameter is **required**. |

## FAQ

### OSB-Redis crashed

If the service broker crashes, the operator should be contacted.

### A Redis master/replica instance crashed

As long as there is at least one replica instance or the master instance running after another instance failed, Redis is still functional. ?Nutzen wir Redis Sentinel für Healthchecks?

The following causes can lead to a failure: ?passt das noch?
- IaaS problems with VMs, network or storage
- Storage space completely occupied
- SSL certificates expired
- Automatic Failover requires Zookeeper

Access to the VM via [Bosh CLI](https://bosh.io/docs/cli-v2/) is required for debugging. The logs of Redis can be acquired within the VM under:

```
/var/vcap/sys/log/redis/?ALL?
```
??hiweis auf redis-cli? unter /var/vcap/packages/redis/bin

If the error **cannot be fixed**, a new instance has to be created and restored by using a backup.

### The size of the backup was bigger than expected (and failed) and now all of my storage space is occupied TODO

In this case, contact the operator.

## Appendix

### JSON-Schema

```
schemas: &schemas
        service_binding:
          create:
            parameters:
              properties:
                database:
                  description: Specify the database for the service binding.
                  type: string
              schema: http://json-schema.org/draft-04/schema#
              type: object
        service_instance:
          update: &schemaproperties
            parameters:
              properties:
                postgres:
                  properties:
                    version:
                      type: string
                      default: "14"
                    ssl:
                      type: object
                      properties:
                        enabled:
                          title: ssl enabled
                          type: boolean
                          default: true
                        min_protocol_version:
                          title: minimal TLS version
                          type: string
                          default: "TLSv1.2"
                          enums:
                          - TLSv1
                          - TLSv1.1
                          - TLSv1.2
                          - TLSv1.3
                          - ""
                        max_protocol_version:
                          title: minimal TLS version
                          type: string
                          default: "TLSv1.3"
                          enums:
                          - TLSv1
                          - TLSv1.1
                          - TLSv1.2
                          - TLSv1.3
                          - ""
                    database:
                      type: object
                      properties:
                        extensions:
                          items:
                          - type: string
                            enums: &extension 
                            - address_standardizer
                            - address_standardizer_data_us
                            - adminpack
                            - amcheck
                            - autoinc
                            - bloom
                            - bool_plperl
                            - bool_plperlu
                            - btree_gin
                            - btree_gist
                            - citext
                            - cube
                            - dblink
                            - dict_int
                            - dict_xsyn
                            - earthdistance
                            - file_fdw
                            - fuzzystrmatch
                            - hstore
                            - hstore_plperl
                            - hstore_plperlu
                            - hstore_plpython2u
                            - hstore_plpython3u
                            - hstore_plpythonu
                            - insert_username
                            - intagg
                            - intarray
                            - isn
                            - jsonb_plperl
                            - jsonb_plperlu
                            - jsonb_plpython2u
                            - jsonb_plpython3u
                            - jsonb_plpythonu
                            - lo
                            - ltree
                            - ltree_plpython2u
                            - ltree_plpython3u
                            - ltree_plpythonu
                            - moddatetime
                            - old_snapshot
                            - pageinspect
                            - pg_buffercache
                            - pgcrypto
                            - pg_freespacemap
                            - pg_prewarm
                            - pgrowlocks
                            - pg_stat_statements
                            - pgstattuple
                            - pg_surgery
                            - pg_trgm
                            - pg_visibility
                            - plperl
                            - plperlu
                            - plpgsql
                            - plpython3u
                            - postgis
                            - postgis_raster
                            - postgis_sfcgal
                            - postgis_tiger_geocoder
                            - postgis_topology
                            - postgres_fdw
                            - refint
                            - seg
                            - sslinfo
                            - tablefunc
                            - tcn
                            - tsm_system_rows
                            - tsm_system_time
                            - unaccent
                            - uuid-ossp
                            - xml2
                          type: array
                    config:
                      properties:
                        authentication_timeout:
                          title: Authentication Timeout
                          type: integer
                          minimum: 1
                          default: 30
                        max_locks_per_transaction:
                          title: Maximum Locks per Transaction
                          type: integer
                          minimum: 10
                          default: 64
                        max_pred_locks_per_page:
                          title: Maximum Predicate Locks per Page
                          type: integer
                          minimum: 1
                          default: 2
                        max_pred_locks_per_relation:
                          title: Maximum Predicate Locks per Relation
                          type: integer
                          default: -2
                        max_pred_locks_per_transaction:
                          title: Maximum Predicate Locks per Transaction
                          type: integer
                          minimum: 10
                          default: 64
                      required:
                      - authentication_timeout
                      - max_locks_per_transaction
                      - max_pred_locks_per_transaction
                      - max_pred_locks_per_relation
                      - max_pred_locks_per_page
                      title: General PostgreSQL Settings
                      type: object
                    databases:
                      items:
                      - properties:
                          name:
                            pattern: ^[A-Za-z0-9_-]+$
                            type: string
                          users:
                            items:
                            - type: string
                            type: array
                          extensions:
                            items:
                            - type: string
                              enums: *extension 
                            type: array
                        required:
                        - name
                        type: object
                      type: array
                    users:
                      items:
                      - properties:
                          admin:
                            type: boolean
                          password:
                            type: string
                          username:
                            pattern: ^[A-Za-z0-9_-]+$
                            type: string
                        required:
                        - username
                        - password
                        type: object
                      type: array
                  title: ProgreSQL Configuration
                  type: object
              schema: http://json-schema.org/draft-06/schema
              type: object
          create: *schemaproperties
```