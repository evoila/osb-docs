# OSB-RabbitMQ
- [OSB-RabbitMQ](#osb-rabbitmq)
  - [Overview](#overview)
    - [Key Features](#key-features)
    - [Software used by OSB-RabbitMQ](#software-used-by-osb-rabbitmq)
    - [Cluster](#cluster)
  - [TODO Requirements](#todo-requirements)
  - [TODO How to](#todo-how-to)
    - [Create a Service Instance](#create-a-service-instance)
    - [Update a Service Instance](#update-a-service-instance)
    - [Create a Service Binding](#create-a-service-binding)
    - [Acquiring Service Instance Parameters](#acquiring-service-instance-parameters)
    - [Backup](#backup)
    - [Change SSL Certificates](#change-ssl-certificates)
  - [Settings](#settings)
    - [Service Instance Settings Schema](#service-instance-settings-schema)
      - [Server object](#server-object)
      - [SSL object](#ssl-object)
    - [Ingress and egress binding](#ingress-and-egress-binding)
  - [TODO FAQ](#todo-faq)
    - [How a can the status of the cluster be viewed?](#how-a-can-the-status-of-the-cluster-be-viewed)
    - [Cluster performance](#cluster-performance)
    - [etc](#etc)
    - [OSB-RabbitMQ crashed](#osb-rabbitmq-crashed)
    - [A Elasticsearch instance crashed (?anwendbar auf rabbitmq?)](#a-elasticsearch-instance-crashed-anwendbar-auf-rabbitmq)
    - [The size of the backup was bigger than expected (and failed) and now all of my storage space is occupied](#the-size-of-the-backup-was-bigger-than-expected-and-failed-and-now-all-of-my-storage-space-is-occupied)
  - [Appendix](#appendix)

---

## Overview

RabbitMQ is a lightweight message broker that can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements. RabbitMQ lets developers control the routing of the messages by using the messages metadata instead of having the message broker administrator define the routes. RabbitMQ is used worldwide at small startups and large enterprises and tens of thousands of users which makes it one of the most popular open source message brokers.

### Key Features

- **Flexible Routing**: Messages are routed through exchanges before arriving at queues. RabbitMQ features several built-in exchange types (direct, topic, fanout and headers) for typical routing logic. For more complex routing you can [bind exchanges together](https://www.rabbitmq.com/extensions.html#routing) or even write your own exchange type as a plugin.
- **Highly Available Queues**: By mirroring queues across several machines in a cluster, messages are safe even in the event of hardware failure. For more information, see [What is Mirroring?](https://www.rabbitmq.com/ha.html#what-is-mirroring).
- **Multi-protocol**: RabbitMQ supports messaging over a variety of messaging protocols. For more information, see [Which Protocols does RabitMQ support?](https://www.rabbitmq.com/protocols.html).
- **Client Libraries for many Languages**: RabbitMQ has several official client libraries. Libraries exist for, among many other programming languages, Java (and Spring), .NET, Ruby, Python, Javascript (and Node.js) and Go. For a complete list, see [Clients Libraries and Developer Tools](https://www.rabbitmq.com/devtools.html).
- **Management UI**: RabbitMQ has an integrated [management UI](https://www.rabbitmq.com/management.html) which allows for monitoring and controlling of the message broker.
- **Tracing**: RabbitMQ has a [firehose](https://www.rabbitmq.com/firehose.html) feature which makes it able to see every message that is published and every message that is delivered so that, for example, unacked messages can be found. 
- **Plugin System**: RabbitMQ ships with a variety of plugins extending it. Plugins can be selected by sending request parameters within a create/update request sent to the broker (see[Settings](#settings)). Additionally, own plugins can be written (?werden Kunde eigene Plugins schreiben? oder lieber raus nehmen?).

A complete list of features can be found [here](https://www.rabbitmq.com/features.html).

For more information, see the [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html).

This project is part of our service broker project. For documentation of the service broker see [evoila/osb-docs](https://github.com/evoila/osb-docs).
The OSB-Elasticsearch offers different service plans which vary in allocated memory, cpu, disc-size and number of vms created for Elasticsearch. (gleich?)

### Software used by OSB-RabbitMQ
- **RabbitMQ**: ???3.6.12

### Cluster



A RabbitMQ cluster can consist of a single node but it is recommended to have at least 3 nodes if high availability is a concern. The master node is called *leader* and undertakes the task of queueing messages while the other nodes are called *mirrors*, which apply the operations that occur to the leader exactly in the same order as the leader and thus maintain the same state. Requests to a RabbitMQ instance are first sent to a HAProxy instance which then redirects the requests to the leader node.

If a mirror fails, no client needs to take any action or be informed about the failure. The cluster will still work and the mirror node will be restarted when the other nodes do not receive a heartbeat in time (the default time for net ticks is 60 seconds).

If the leader fails, the longest running mirror is promoted to leader. If no mirror is completely synchronized to the leader, messages that only existed on the leader will be lost.

> **_IMPORTANT:_** The amount of master-eligible nodes together with the general nodes must be odd. (?odd wird hier gar nicht gebraucht, da einfach der longest running mirror ausgewählt wird, oder?)

(?Bild anpassen? Kasten "RabbitMQ Service Instance" um Cluster und HAProxy?)
The following image shows, how RabbitMQ nodes are managed:
![RabbitMQ cluster](../assets/rabbitmq-cluster.png)

## TODO Requirements
- [Cloud Foundry CLI](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)
- Memory: A node should have at least 4 GB of RAM (recommended minimum is 8 GB) but not greater than 64 GB (see [Heap: Sizing and Swapping](https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html))
- Storage: For memory-intense search workloads a ram to storage workload of 1:16..(?TODO)

## TODO How to
### Create a Service Instance

A service instance can be created manually via the CLI-Command
```
cf create-service SERVICE PLAN SERVICE_INSTANCE [-b BROKER] [-c PARAMETERS_AS_JSON] [-t TAGS]
```

- **SERVICE** will be the name of the service broker which is likely going to be **osb-rabbitmq**
- **PLAN** is a plan offered by the service.
- **SERVICE_INSTANCE** the name of the service instance, can be chosen freely.
- **PARAMETERS_AS_JSON** contains additional parameters in JSON-format.

For more information see [Cloud Foundry CLI Reference Guide](https://cli.cloudfoundry.org/en-US/v6/create-service.html)

?Zeigt dashboard parameter? Info raus?Erzeugen geht ja theoretisch
Aternatively, if there is a dashboard set up (like the Stratos Dashboard for example), it can be used to create a service instance.

### Update a Service Instance

A service instance can be updated manually via the CLI-Command
```
cf update-service SERVICE_INSTANCE [-p NEW_PLAN] [-c PARAMETERS_AS_JSON] [-t TAGS] [--upgrade]
```

- **SERVICE_INSTANCE** is be the name of the previously created service instance.
- **PARAMETERS_AS_JSON** contains additional parameters in JSON-format.

For more information see [Cloud Foundry CLI Reference Guide](https://cli.cloudfoundry.org/en-US/v6/update-service.html)

?Zeigt dashboard parameter? Info raus?Erzeugen geht ja theoretisch
Aternatively, if there is a dashboard set up (like the Stratos Dashboard for example), it can be used to update a service instance.

> **_IMPORTANT:_** Keep in mind that **previous values will be overwritten**. In order to see the existing parameters you can use a dashboard or acquire the parameters via cli (see [Acquiring Service Instance Parameters](#acquiring-service-instance-parameters)).
### Create a Service Binding

A binding can be created manually via the CLI-Command 
```
cf bind-service APP_NAME SERVICE_INSTANCE [--binding-name BINDING_NAME]
```

- **APP_NAME** ist the name of the previously created app that gets the binding injected.
- **SERVICE_INSTANCE** is be the name of the previously created service instance.
(?parameters werden nicht gebraucht, da kein schema für service_binding)


For more information see [Cloud Foundry CLI Reference Guide](https://cli.cloudfoundry.org/en-US/v6/bind-service.html).
After creating a binding, the app has to be restarted for the changes to take effect. (??muss rabbitmq service dann neu gestartet werden?)

?Falls bei create und update parameter nicht gehen: trotzdem auf dashboard bei create binding verweisen?
Aternatively, if there is a dashboard set up (like the Stratos Dashboard for example), it can be used to create a service binding.

### Acquiring Service Instance Parameters

The current parameters of a service instance can be retrieved via cli:

1. ```cf service --guid **SERVICE_INSTANCE**```
2. ```cf curl v3/service_instances/**SERVICE_INSTANCE_ID**/parameters```
3. A JSON with the parameters will be returned.

- **SERVICE_INSTANCE** is be the name of the previously created service instance.
- **SERVICE_INSTANCE_ID** is the guid of the service instance which is acquired in step 1. 


### Backup

TODO 

??Backup

Before creating a backup, a backup client must be set up, which can be done by sending [settings](#settings) while creating/updating a service instance.
Afterwards a snapshot can be triggered via the Elasticsearch API:
```
PUT /_snapshot/my_repository/my_snapshot
```
- **my_repository** is the name of the client.
- **my_snapshot** is the name of the snapshot. **Must** be lowercase.
(wird beim erzeugen vom repo compression definiert?)

TODO END FUTURE

Further information about using the Elasticsearch API to create a snapshot can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/create-snapshot-api.html)

It is important to keep in mind that there is enough space for Elasticearch data and backups. The size of the backup corresponds to the size of the indices (?check compression?) which can be checked by sending a request to the Elasticsearch API (see [Index Stats API](https://www.elastic.co/guide/en/elasticsearch/reference/7.7/indices-stats.html)):
```
GET /_stats
```

The size of a plan can be scaled up afterwards but this will copy the disk.

### Change SSL Certificates

The certificates expire after 365 day. If a certificate is about to expire, contact the operator of the Service Broker to renew the certificates.

If Bosh DNS is used, the certificates are stored in Credhub and can be renewed there. If the root CA is still valid, the certificate can simply be deleted and the new certificate can be used via `bosh manifest`and `bosh deploy`. If the root CA expires, it is necessary to concatenate old and new certificates, for example via [https://github.com/pivotal/credhub-release/blob/main/docs/ca-rotation.md](https://github.com/pivotal/credhub-release/blob/main/docs/ca-rotation.md).</br>
If the IP variant is used and the root CA still valid, it is sufficient to use `bosh recreate`. For changing the root CA, it also has to be concatenated and multiple deploys have to be made.

## Settings

This section covers different settings that can be made for the OSB-RabbitMQ and how they can be changed.
Via the settings, multiple server settings can be changed, for example the plugins used by RabbitMQ and SSL settings.

Settings can be sent as parameters of a create/update request of a service instance via CLI.

The CLI command will look like this:
```
cf cs BROKERNAME PLAN SERVICENAME [-c PARAMETERS_AS_JSON]
```
or
```
cf update-service SERVICE_INSTANCE [-c PARAMETERS_AS_JSON]
```
> **_IMPORTANT:_** Keep in mind that **previous values will be overwritten** when updating. In order to see the existing parameters you can use a dashboard or acquire the parameters via cli (see [Acquiring Service Instance Parameters](#acquiring-service-instance-parameters)).
- **BROKERNAME** will be the name of the service broker which is likely going to be **osb-rabbitmq**
- **PLAN** is the plan that is going to be used for the service instance
- **SERVICENAME** is the name of the service which is up to the user
- **PARAMETERS_AS_JSON** are the settings which are sent in json format

For example, a cli command for creating a service instance could look like this:
```
cf cs osb-elasticsearch s elasticsearch-test -c '{"rabbitmq": {"disk_free_limit": 25000000, "plugins": ["rabbitmq_management", "rabbitmq_delayed_message_exchange"], "ssl": {"enabled": true}}}'
```
An extended example of the parameters for a create/update request for a service instance is shown below:
```json
{
  "rabbitmq": {
    "server": {
      "reverse_dns_lookup": false,
      "disk_alarm_threshold": "{mem_relative,0.4}",
      "disk_free_limit": 50000000,
      "plugins": ["rabbitmq_management", "rabbitmq_delayed_message_exchange"],
      "handshake_timeout": 	10000,
      "net_ticktime": 130,
      "num_ssl_acceptors": 10,
      "frame_max": 131072,
      "channel_max": 2047,
      "cluster_partition_handling": "autoheal",
      "num_tcp_acceptors": 10,
      "fd_limit": 65536,
      "vm_memory_high_watermark": 0.4,
      "ssl": {
        "fail_if_no_peer_cert": false,
        "handshake_timeout": 5000,
        "verify": "verify_none",
        "enabled": true
      }
    }
  }
}
```

In the following section, the fields will be described.

### Service Instance Settings Schema

The following settings are defined in the schema in service_plan.schemas.service_instance.**create**.parameters.properties.rabbitmq.properties and service_plan.schemas.service_instance.**update**.parameters.properties.rabbitmq.properties (?überprüfen ob korrekt)

> **_IMPORTANT:_** If SSL has been enabled via settings, it must not be disabled, while the service instance ist running.?

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| server | [Server](#server-object) object| - | Contains all settings for the RabbitMQ service instance. |


#### Server object

The Server object contains the settings for the RabbitMQ service instance and consists of the following properties:

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| reverse_dns_lookup | boolean | false | Perform reverse DNS lookups when accepting a connection. RabbitMQctl will then display hostnames instead of IP addresses. |
| disk_alarm_threshold | string | {mem_relative,0.4} | The threshold in bytes of free disk space at which rabbitmq will raise an alarm. "{mem_relative}" is relative to the RAM in the machine. Otherwise "absolute" can be used with optional units (e.g. "KB", "MB", "GB") behind an integer number. |
| disk_free_limit | integer | 50000000 | Lower bound for the disk after which a disk alarm will be set. |
| plugins | array of strings | - | Plugins to be used by RabbitMQ. Valid values are "rabbitmq_management", "rabbitmq_mqtt", "rabbitmq_stomp", "rabbitmq_auth_mechanism_ssl", "rabbitmq_delayed_message_exchange". **"rabbitmq_management"** and **"rabbitmq_delayed_message_exchange"** are **required**. |
| handshake_timeout | integer | 10000 | Maximum amount of time allowed for the AMQP 0-9-1 and AMQP 1.0 handshake. |
| ssl | [SSL](#ssl-object) object | - | The SSL object contains the configuration for SSL. |
| net_ticktime | integer | 130 | Time until a node is considered lost if no heartbeat is sent. |
| num_ssl_acceptors | integer | 10 | Number of Erlang processes that will accept connections for the TLS. |
| frame_max | integer | 131072 | Set the max permissible size of an AMQP frame (in bytes). |
| channel_max | integer | 2047 | Max permissible number of channels per connection. |
| cluster_partition_handling | string | - | Cluster partition recovery mode. Valid values are "pause_minority" and "autoheal". |
| num_tcp_acceptors | integer | 10 | Number of Erlang processes that will accept connections for the TCP. |
| fd_limit | integer | 65536 | The file descriptor limit for the RabbitMQ process. |
| vm_memory_high_watermark | number | 0.4 | Fraction of the high watermark limit at which queues start to page message out to disc in order to free up memory. |

#### SSL object

| Parameter | Type | Default Value | Description |
| - | - | - | - |
| fail_if_no_peer_cert | boolean | - | Sets, whether RabbitMQ server should reject connection if there is no peer cert. |
| handshake_timeout | integer | 5000 | TLS handshake timeout, in milliseconds. |
| verify | string | verify_none | SSL Peer Verification (use 'verify_peer' or 'verifiy_none' to unable/disable). |
| enabled | boolean | false | Enable SSL/TLS. |

### Ingress and egress binding

?WEG
When you create a new service binding in cloudfoundry, you can provide an client mode, either ingress or egress. The broker automatically filters the nodes and returns only the IPs corresponding to the client mode passed. If you do not specify a client mode, egress will be used.

Example:

`cf bind-service APP_NAME SERVICE_INSTANCE -c '{"clientMode":"ingress"}'`

`cf bind-service APP_NAME SERVICE_INSTANCE -c '{"clientMode":"egress"}'`

## TODO FAQ

### How a can the status of the cluster be viewed?
Generelle Cluster Infos um Rückschlüsse ziehen zu können
### Cluster performance

### etc

### OSB-RabbitMQ crashed

If the service broker crashes, the operator should be contacted.

### A Elasticsearch instance crashed (?anwendbar auf rabbitmq?)

As long as the majority of MongoDB instances is running after another instance failed, the database is still functional. The Bosh director will detect the failure and try to repair the broken instance. If an automatic repair of the instance is not possible, it has to be fixed manually.

The following causes can lead to a failure:
- IaaS problems with VMs, network or storage.
- Storage space completely occupied.

Access to the VM via [Bosh CLI](https://bosh.io/docs/cli-v2/) is required for debugging. First, the following two files have to be sourced within the VM.

```
/var/vcap/jobs/mongodb/env
/var/vcap/jobs/mongodb/bin/post-deploy
```
Start the Mongo cli ($MONGO) as admin (password can be found in **post-deploy** or CredHub for example) --authenticationDatabase admin --host $MONGODB_HOST.

For example:
```
$MONGO -u admin -p ADMIN_PASSWORD --authenticationDatabase admin --host $MONGODB_HOST
```
Afterwards the state of the cluster can be retrieved with the command `rs.status()`. 
Additional information can be retrieved with the command `rs.isMaster()`.

**Majority** of nodes (including primary) failed and can't be reinitialized:</br>
In this case no secondary can be elected as primary. The only options are reinitializing the replica set or [reconfiguring](https://docs.mongodb.com/manual/tutorial/reconfigure-replica-set-with-unavailable-members/) it.

In many cases it is also possible, to retrieve the oplog of a failed primary with [`mongodump`](https://docs.mongodb.com/database-tools/mongodump/) using the `--oplog` flag and recover the database by replaying the oplog with [`mongorestore`](https://docs.mongodb.com/database-tools/mongorestore/) using the `--oplogReplay` flag.
Currently, replaying an oplog is not supported by the Backup Manager. Changes to the database while doing a backup **may not** be recorded.

The following logs are relevant for troubleshooting:

```
/var/vcap/sys/log/mongodb/server.stderr.log
/var/vcap/sys/log/mongodb/server.stdout.log
```

While performing maintenance on single replica set members, each node has to be restarted as standalone. Afterwards, maintenance can be done on the node and then be restarted as a member of the replica set. The primary should come last. More information about maintaining replica set members can be found [here](https://docs.mongodb.com/manual/tutorial/perform-maintence-on-replica-set-members/).

If the error **cannot be fixed**, a new instance has to be created and restored by using a backup.

### The size of the backup was bigger than expected (and failed) and now all of my storage space is occupied

In this case, contact the operator.

## Appendix

(?nur schema? schema hier? kompletter catalog in assets? -> weitere anpassungen im catalog außer persisten disk und vm type?)