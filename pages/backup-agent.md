# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](service-keys.md)
5. [Backup Agent](#backup-agent)
    * 5.1 [Configuration in the Service Definition](#configuration-in-the-service-definition)
    * 5.2 [Implementation of the Backup Connector](#implementation-of-the-backup-connector)
6. [Development](development.md)
7. [Open Service Broker API Features](osb-api-features.md)
8. [IDE & Runtime](ide-runtime.md)
9. [Contribution](contribution.md)
10. [License](license.md)
---

# Backup Agent
The Backup Manager is a component provided with the evoila OSB Framework in the Enterprise Package. It enables to schedule automatic backups of any endpoint. That means,each endpoint that has a concrete implementation in the `osb-backup-manager` enables customers/developers to do automatic backups of their Service Instances. 


## Configuration in the Service Definition
To enable the Backup Connector configuration wise, the Service Broker needs to be deployed with the following settings:

```yaml
backup:
  enabled: true
catalog:
  services:
    plans:
      - name: xs
        ...
        metadata:
          backup:
            enabled: true
            instance_group: postgres
```
* **enabled**: if set to true it is enabled for the specific plan.
* **instance_group**: the instance group the osb-backup-manager is connection to, to execute the backups.

*Note: please be aware, that the communication to schedule the backup jobs is done via RabbitMQ. That means, an existing RabbitMQ configuration must exist. Refer to Basic Service Broker configuration for that.*

## Configuration in the BoshPlatformService
To set the backup functionality for the instance group, configured in the metadata of a plan, use the method  
```java
protected ServerAddress toServerAddress(Vm vm, int port, Plan plan)
```
Example:
```java
@Service
@ConditionalOnBean(BoshProperties.class)
public class MyBoshPlatformService extends BoshPlatformService {

    @Override
    protected void updateHosts(ServiceInstance serviceInstance, Plan plan, Deployment deployment) {
        List<Vm> vms = super.getVms(serviceInstance);
        if (serviceInstance.getHosts() == null)
            serviceInstance.setHosts(new ArrayList<>());
        serviceInstance.getHosts().clear();

        vms.forEach(vm -> serviceInstance.getHosts().add(super.toServerAddress(vm, 1234, plan)));
    }
}
```

## Implementation of the Backup Connector
The Backup Connector is a component 
```java
@Service
@ConditionalOnBean(BackupTypeConfiguration.class)
public class BackupCustomServiceImpl implements BackupCustomService {

    BackupTypeConfiguration backupTypeConfiguration;

    ServiceInstanceRepository serviceInstanceRepository;

    PostgresCustomImplementation postgresCustomImplementation;

    ServiceDefinitionRepository serviceDefinitionRepository;

    public BackupCustomServiceImpl(BackupTypeConfiguration backupTypeConfiguration,
                                   ServiceInstanceRepository serviceInstanceRepository,
                                   PostgresCustomImplementation postgresCustomImplementation,
                                   ServiceDefinitionRepository serviceDefinitionRepository) {
        this.backupTypeConfiguration = backupTypeConfiguration;
        this.serviceInstanceRepository = serviceInstanceRepository;
        this.postgresCustomImplementation = postgresCustomImplementation;
        this.serviceDefinitionRepository = serviceDefinitionRepository;
    }

    @Override
    public Map<String, String> getItems(String serviceInstanceId) throws ServiceInstanceDoesNotExistException,
            ServiceDefinitionDoesNotExistException {
        ServiceInstance instance = this.validateServiceInstanceId(serviceInstanceId);

        Plan plan = serviceDefinitionRepository.getPlan(instance.getPlanId());

        PostgresDbService postgresDbService = postgresCustomImplementation.connection(instance, plan);

        Map<String, String> result = new HashMap<>();
        try {
            Map<String, String> databases = postgresDbService.executeSelect("SELECT datname FROM pg_database", "datname");

            for(Map.Entry<String, String> database : databases.entrySet())
                result.put(database.getValue(), database.getValue());
        } catch(SQLException ex) {
            new ServiceBrokerException("Could not load databases", ex);
        }

        return result;
    }
}
```

---

<p align="center">
    <span ><a href="service-keys.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="development.md">Next -></a></span>
</p>
