# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](service-keys.md)
5. [Backup Agent](backup-agent.md)
6. [Development](#development)
    * 6.1 [Bosh](#bosh)
      * 6.1.1 [Cloud Config](#cloud-config)
    * 6.2 [Backends](#backends)
      * 6.2.1 [MongoDB](#mongodb)
      * 6.2.2 [RabbitMQ](#rabbitmq)
7. [IDE & Runtime](ide-runtime.md)
8. [Contribution](contribution.md)
9. [License](license.md)
---

# Development

## Bosh

### Cloud Config
Please apply the following `cloud-config` to your environment via `bosh update-cloud-config cloud-config.yml`.

```yaml
azs:
- name: z1
- name: z2
- name: z3
compilation:
  az: z1
  network: default
  vm_type: default
  workers: 3
disk_types:
- disk_size: 1024
  name: default
- disk_size: 3000
  name: small
- disk_size: 20000
  name: medium
- disk_size: 60000
  name: large
- disk_size: 100000
  name: xlarge
- disk_size: 5000
  name: 5GB
- disk_size: 10000
  name: 10GB
- disk_size: 100000
  name: 100GB
- disk_size: 1000000
  name: 1000GB
networks:
- name: default
  subnets:
  - azs:
    - z1
    - z2
    - z3
    dns:
    - 8.8.8.8
    range: 10.244.0.0/24
    reserved:
    - 10.244.0.1
    - 10.244.0.2
  type: manual
- name: service
  subnets:
  - azs:
    - z1
    - z2
    - z3
    dns:
    - 8.8.8.8
    range: 10.245.0.0/24
    reserved:
    - 10.245.0.1
    - 10.245.0.2
  type: manual
- name: floating
  type: vip
vm_types:
- cloud_properties:
    ephemeral_disk:
      size: 1024
      type: gp2
  name: default
- name: compile
- name: minimal
- name: small
- name: medium
- name: large
```

## Backends
### MongoDB
### RabbitMQ

--- 

<div style="overflow: hidden;">
    <p style="float: left;"><a href="backup-agent.md"><- Previous</a></p>
    <p style="float: right;"><a href="ide-runtime.md">Next -></a></p>
</div>