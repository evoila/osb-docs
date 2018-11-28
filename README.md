
# Open Service Broker Framework (by evoila)

A Service Broker provides the possibility to extend a platform [Cloud Foundry](https://www.cloudfoundry.org/) and [Kubernetes](https://www.https://kubernetes.io//) with services (for example a database) that can be consumed by applications deployed to this platform.

![](assets/service_broker_1.png)

## What does that mean in detail?
When deploying applications to Container as a Service (CaaS) or Platform as a Service (PaaS), applications need to consume backend services like databases, queueing engines.  
  
To standardize the consumption and usage of backend service the [OSB-API](https://github.com/openservicebrokerapi/servicebroker/) spec define a set of operations,which are describe by the following terms:
* Catalog
    * Services 
    * Plans
* Service Instances (a database in a Cluster/a complete dedicated database cluster)
* Service Bindings (credentials username/password, ceritificates dynamically created and provided for specific application)

![](assets/service_broker_2.png)

For a better understanding of Cloud Foundry Service Brokers also visit [https://docs.cloudfoundry.org/services/api.html](https://docs.cloudfoundry.org/services/api.html).

# Terminology of the evoila OSB-Framework

## Shared Clusters (Existing Service)
Shared Clusters host Service Instances of several users with a multitenancy separation. E.g. a large PostgreSQL Cluster, which allows customers/developers to provision a database instance. 

Instances on shared clusters are particularly inexpensive, but have limitations in their performance and parameterization possibilities. The services should be hardened so that users can never access external data. Shared Cluster and Service Instances on it should NEVER BE USED FOR PRODUCTION. Most databases do not provide out-of-the-box features for noisy neighbours, connection and resource limitation on a granular level.

As you can't scale clusters indefinitely or you want to provide different service classes of your existing clusters (Silver, Gold, Platinum based on Storage Performance), the OSB Framework provides the ability to configure an unlimited number of so called `existing endpoints`. 

For more information see the documentation regarding our declarative approach of Plans. 

## Dedicated VMs/Clusters
Dedicated VMs/Containers and Clusters are Service Instances that are provisioned as needed. Usually the customer is granted more rights on these Service Instances, e.g. to create more databases or users. In addition, more parameters of the service can be set to adapt performance to his use case.  

## Site
A Site is a dedicated Cloud Foundry/Kubernetes deployment. The Open Service Broker Framework currently supports an unlimited number of parallel sites.

# Components
In the OSB Framework we speek of components as submodules, which are sliced to fullfil a specific task. On the development level these are things like providing the OSB REST API , persiting data to a database and deploying Service Instance with an automation technology. 

Currently supported OSB-API version: v2.14

## Core
### Core
In core we separate between implemented features which are 100% compliant with OSBAPI Spec and can be found here: [OSB-API](https://github.com/openservicebrokerapi/servicebroker/) and the custom implementations we created to extend the OSB-API spec at edges/features requirement we repeatedly see at customer sides but are not covered by the OSB-API spec yet.  
Besides the controllers, and service to provision/bind (synch and asynch), core also contains the interfaces definitions, which are implemented in the persistence and deployment modules.   
This generic implementation enables Service Broker developers to use different storage backends for Service Instances, Service Bindings and also different deployment implementations like Bosh, Helm, Terraform or others.

### Model
The Model Project contains alls data classes shared accross all Service Brokers. Like [Configuration Propierties](#configuration), [Service Catalog](#catalog), as well as Request and Response classes.

### Security
The security projects contains a reference implementation for a OpenId Connect authentication, which is the reference authentication mechanism, when using a Dashboard that communicates with the Service Broker backend or other endpoints.

Additionally it provides and interface to store:
* Credentials
* Random Keys
* Certificates

in Credhub. 

## Persistence
Implements the persistence interface with services using MongoDB for Service Brokers. So the management data and status of the service brokers can be persisted in the aforementioned database. Its implemented with Spring Data. This only needs changes if you add new data that needs to be persisted or you want to add a new persitence layer.  

## Deployment

### General
Generally available methods for each Deployment Backend:

#### Create Instance
- `preCreateInstance` to set some things up beforehand
- `postCreateInstance` to add some additional information or execute tasks

#### Update Instance
- `preUpdateInstance` to set some things up beforehand
- `postUpdateInstance` to add some additional information or execute tasks

#### Delete Instance
- `preDeleteInstance` to set some things up before deletion of the service
- `postDeleteInstance` to do some additional cleanup, after the deployment was deleted
- all other methods you only change if you really need to.

### Existing-Service
Existing Service implements what was covered in the section [shared clusters](#shared_cluster). From an implementation perspective this is the most streamlined and easy implementable solution for a developer. 

E.g. connection to an existing PostgreSQL Cluster and creating a new database in the cluster. 

### Bosh
The Bosh module of the Deployment project manages [dedicated clusters](#dedicated_cluster) deployed via Bosh. The main classes you want to inherit from or change are `BoshPlatformService.java` and `DeploymentManager.java`.   
The PlatformService manages is your hook into the lifecycle of the Service Instance. Possible hooks you can override if your Service Broker needs to, are:   

#### Invocation of Bosh functionality
- `runCreateErrands` to run errands after a service instance was deployed sucessfully
- `runUpdateErrands` to run errands after a service instance was updated sucessfully
- `runDeleteErrands` to run errands before deleting a service instance
- `updateHosts` you need to update the hosts list for your deployment. *IF YOUR DONT DO THIS PROVISIONING WILL FAIL*

The `DeploymentManager` manages the parameterization of your Bosh deployment. It loads a default manifest from `$resource_path/bosh/manifest.yml` that needs to be provided by you. You propably want also to hook into `replaceParameters` which allows you change parameters on the deployment manifest during create or update of a Service Instance.

### Openstack (Deprecated)
Due to a significant amount of problems with OpenStack Heat, not maintained anymore. This project is deprecated and will be removed.

## Dashboard
The dashboard is a Single Page Application (SPA) with different modules for each Service Broker. When you want to create a new Dashboard module for your specific Service Broker the first step is (after you have made yourself comfortable with Angular 5+ and reviewed all the shared components the Dashboard brings along) to create a new entry in: `.angular-cli.json`, which should looks as follows:

```json
{
    "name": "new-module", #(any fancy DB you want to Service Brokerize)
    "root": "src",
    "outDir": "dist/example",
    "deployUrl": "/app/",
    "assets": [
        "assets/core",
        "favicon.ico"
    ],
    "index": "index.html",
    "main": "main.ts",
    "polyfills": "polyfills.ts",
    "test": "test.ts",
    "tsconfig": "tsconfig.app.json",
    "testTsconfig": "tsconfig.spec.json",
    "prefix": "sb",
    "styles": [
        "styles.scss"
    ],
    "stylePreprocessorOptions": {
        "includePaths": [
            "../node_modules"
        ]
    },
    "scripts": [],
    "environmentSource": "environments/target.ts",
    "environments": {
    "new-module": "environments/target.new-module.ts"
    }
}
```
In the next step you should be able to configure `package.json` and add the necessary entries for start, build and ci steps. 

```javascript 
"scripts": {
    ...
    "start:new-module": "ts-node ./scripts/start.ts -t new-module",
    ...
    "build:new-module": "ts-node ./scripts/build.ts -t new-module",
    ...
    "e2e:new-module": "ts-node ./scripts/dist-e2e.ts -t new-module",
    ...
    "ci:new-module": "yarn run tslint && yarn run test && yarn run build:new-module && yarn run e2e:new-module",
}
```

The last step is to create an environment file in `src/app/environments`, which defines the default modules being used and the location of the new module for the specific Service Broker:

```javascript
import { BuildTarget } from './build-target';

/**
 * We keep auth module separate from Meshcloud module so we can have diverging
 * set of meshstack submodules (e.g. billing/profile/register) between VW and meshcloud targets
 */
export const buildTarget: BuildTarget = {
  coreModules: [
  ],
  sharedModules: {
    // The modules define which menu entries should be enabled by default
    general: true,
    // general: just provides an overview regarding Service Instance Details
    backup: false,
    // enables the integrated OSB Backup Framework Dashboard
    management: true,
    // enables the view to list/create items for your Service Instance Backend. E.g. in context of PostgreSQL within a dedicated cluster creating/listing databases
    serviceKeys: true
    // a feature mainly dedicated to Cloud Foundry, which enables customers to access Service Instances through an SSH tunnel 
  },
  extensionModules: [
    // https://angular.io/docs/ts/latest/guide/router.html#!#preload-canload
    //  If you want to preload a module and guard against unauthorized access, drop the canLoad guard and rely on the CanActivate guard alone.
    {
      path: 'example',
      loadChildren: 'app/example/example.module#NewModuleModule', // yes the redundancy is in intended
      data: {
        skipPreload: true // register module is seldomly needed, save these few kbs
      }
    }
  ]
};
```
After having defined those files, you can follow the `README.md` in the [Dashboard repository](https://github.com/evoila/osb-dashboard).

# Configuring a Service Broker

# Basic Configuration
Within the following sections the documentation covers the maximum number of possibilities in regard of Service Broker configuration for the evoila OSB Framework. Each configuration section, which is not mandatory, is marked with (Optional).

In the `spring` section we define the endpoint properties, for communication between Config Server, Service Key Manager, Backup Manager and Service Broker (RabbitMQ) and the storage backend which is based on MongoDB. In regard to the storage backend it is important to note, that MongoDB might be replaced with any database of choice. The only thing that needs to adapted are the interfaces in the `osb-persistence` module.

*Note: an important note here is, that we correlate multi site deployments with spring profiles. Means, the profiles section contains the name of the site, where the Service Broker is deployed.*

```yaml
spring:
  profiles: site1
  data:
    mongodb:
      database: dbName
      host: host1
      username: username1
      password: password1
      port: 27017        
  rabbitmq:
    host: host1
    username: usernam1
    password: password1
    port: 5672    
    virtual-host: vhost1
  thymeleaf:
    cache: true
    mode: LEGACYHTML5
```
The basic configuration for communication with the backend database, RabbitMQ (Config Updates, Service Key Creation) and Thymeleaf for Service Broker dashboard delivery. 

```yaml
login:
  password: cloudfoundry
  role: USER
  username: admin
```
The login section configures username and password (for Basic Authentication), which need to be provided, when the Service Broker is registered in Cloud Foundry or Kubernetes.

As defined in the Catalog (see below), the Open Service Broker Framework provides multiple deployment procedures to be configured within in one Service (separated through Plans). 

The following three configurations show:
* Bosh
* Existing Endpoint
* Openstack

**Bosh**
```yaml
bosh:
  authentication: OAUTH
  host: host1
  username: username1
  password: password1
  stemcellOs: ubuntu-trusty
  stemcellVersion: 3468.13  
  vip_network: floating1
```

While host, username and password a self-explanatory, we provide a definition for the other properties:
* authentication (BASIC for Basic Authentication, OAUTH for OAuth2 Authentication)
* stemcellOs: a "Bosh-global" definition of the default Stemcell
* stemcellVersion: a "Bosh-global" definition of the default Stemcell Version
* vip_network: in Bosh you can define VIP networks for Floating IPs (e.g. used in a scenario where a dedicated Load Balancer needs a Public IP)

This config is NOT mandatory if you don't plan any interaction with Bosh for deployment/automation purposes. 

**OpenStack**
```yaml
openstack:
  cinder:
    az: zone00
  endpoint: https://os.domain:5000/v3
  imageId: 7e72a4b2-8dd4-4a3c-94f9-0ad895b7a622
  keypair: keypair-name
  networkId: feb96c822-631f-483f-a4b3-8eb4e87be9e9
  pool: publicIpPoolName
  project:
    domainName: domainName
    projectName: projectname
  subnetId: 9fe94282-36e8-42a9-8af9-fac6b68a8957
  user:
    domainName: domainName
    username: username1
    password: password1    
```

OpenStack properties should be pretty straight-forward and well known for OpenStack Users. 

This config is NOT mandatory if you don't plan any interaction with OpenStack for deployment/automation purposes. 

**Existing Endpoints** 
```yaml
existing:
  endpoints:
    - server_name: platinum_cluster
      hosts: 
      - name: mongodb1
        ip: 172.16.105.3
        port: 27017
      - name: mongodb2
        ip: 172.16.105.4
        port: 27017
      - name: mongodb4
        ip: 172.16.105.5
        port: 27017      
      database: adminDatabase
      username: username1
      password: password1
    - server_name: gold_cluster
      hosts: 
      - name: mongodb1
        ip: 172.16.105.3
        port: 27017
      - name: mongodb2
        ip: 172.16.105.4
        port: 27017
      - name: mongodb4
        ip: 172.16.105.5
        port: 27017 
      database: adminDatabase
      username: username1
      password: password1
```
This section is a bit longer. It contains the definition for two existing endpoints. Important here is to define unique `server_name` and all the machines, which are needed to be contacted during service/binding creation, as well as the username/password and authentication database. 

## Catalog Configuration
The Catalog definition is an important part of the Service Broker deployment infrastructure. To demonstrate the implemented capabilites, we use the example of the `osb-lbaas` Catalog definition to show it capabilities: 
```yaml
catalog:
  services:
  - bindable: false
    dashboard:
      auth_endpoint: https://uaa.cf.domain.msh.host/oauth
      url: https://osb-lbaas.cf.domain.msh.host/v2/authentication
    dashboard_client:
      id: osb-lbaas.cf.domain.msh.host
      redirect_uri: https://osb-lbaas.cf.domain.msh.host/v2/authentication
      secret: DA2EDFE6-130C-4353-90EE-C202F2BD40F9
    description: LbaaS Instances
    id: 9F13ADE7-0AAC-468D-807F-3BD937FE530D
    name: osb-lbaas
    plans:
    - description: A simple LbaaS Bosh Deployment plan.
      free: false
      id: BB0DD792-C405-411C-B527-9771FE34B2D9
      metadata:
        connections: 1000
        instance_group_config:
        - name: haproxy
          properties:
            ha_proxy:
              stats_enable: false
              stats_password: me
              stats_user: overwrite
        networks:
        - default:
          - dns
          - gateway
          name: service
        - name: floating
        nodes: 1
        vm_type: minimal
      name: s
      platform: BOSH
    - description: A simple LbaaS Bosh Deployment plan.
      free: false
      id: CB0EE792-C405-411C-B527-9771FE53D329
      metadata:
        connections: 1000
        networks:
        - default:
          - dns
          - gateway
          name: service
        - name: floating
        nodes: 1
        vm_type: small
      name: m
      platform: BOSH
```
In the following sections we will explain the specifc details of a Catalog step by step. 

### Dashboard Client
When you are using the Dashboard and Core components of the OSB-Framework, the defintion of `url` and `redirect_uri` always need to point to `/custom/v2/authentication`, which is the endpoint for authentication and the delivery of the dashboard. Important here to note is, that the `id` should be multisite aware and the secret might be a UUID as seen below.
```yaml
...
    dashboard:
      auth_endpoint: https://uaa.cf.domain.com/oauth
      url: https://osb-lbaas.cf.domain.com/custom/v2/authentication
    dashboard_client:
      id: osb-lbaas.cf.domain.com
      redirect_uri: https://osb-lbaas.cf.domain.com/custom/v2/authentication
      secret: DA2EDFE6-130C-4353-90EE-C202F2BD40F9
```

### Plans
The Plan definition is according to the Open Service Specification with extension made b the OSB Framework to provide a delcarative approach of feature enrichment on a plan level. This declarative approach enables Service Catalog managers, which are not necessarily developers to change/extend the Service Catalog without any coding. 

All the important configurations you can make and are still compliant the the OSB-API standard are located in the metadata section of a Plan.
```yaml
...
  plans:
  - name: s
    platform: BOSH
    description: A simple LbaaS Bosh Deployment plan.
    free: false
    id: BB0DD792-C405-411C-B527-9771FE34B2D9
    metadata:
      instance_group_config:
      - name: haproxy
        properties:
          ha_proxy:
            stats_enable: false
            stats_password: me
            stats_user: overwrite
      networks:
      - default:
        - dns
        - gateway
        name: service
      - name: floating
      nodes: 1
      vm_type: minimal
    schemas:
        service_binding:
          create:
            parameters:
              properties:
                admin:
                  description: is an admin
                  type: boolean
                connections:
                  description: connections
                  type: number
                database:
                  description: Specify the database for the service binding.
                  type: string
                list:
                  description: a simple list
                  type: array
              schema: http://json-schema.org/draft-04/schema#
              type: object
        service_instance:
          create:
            parameters:
              properties:
                mail:
                  description: mail for reference
                  pattern: ^[_a-z0-9-]+(.[a-z0-9-]+)@[a-z0-9-]+(.[a-z0-9-]+)*(.[a-z]{2,4})$
                  type: string
                name:
                  description: name of the db that will be created
                  maxLength: 7
                  minLength: 3
                  type: string
                value:
                  description: a value
                  maximum: 10
                  minimum: 2
                  type: integer
              schema: http://json-schema.org/draft-04/schema#
              type: object
...
```

#### Platform
The Platform in the plan section defines, which type of deployment should be called, when the Plan is requested. Currently the OSB Framework offers three different ways of deploying and instance:

* EXISTING_FACTORY
* BOSH
* OPENSTACK (Deprecated)

Depending on the Platform that is configured there, the Service Broker needs to provide a specific implementation for it and automically searches for it. If the specific platform is not implemented the Service Broker throws an exception. 

If there is an implementation, the `PlatformService` picks it up and provides all the information configured in the Plan Metadata.

#### Metadata
The Metadata section provides a fully customized section in the Plan definition of the Service Broker. We will go through the sub-sections one by one:

**Configuration for all VMs**
```yaml
nodes: 1
vm_type: minimal
persistent_disk_type: 10GB
networks:
  - default:
    - dns
    - gateway
    name: service
  - name: floating
custom_parameters:
  property_one: value_one
```

* **nodes**: overwrites the number of instances for *all* VMs in *all* Instance Groups of the Bosh Deployment Manifest.
* **vm_type**: overwrites the VM Type for *all* VMs in *all* Instance Groups of the Bosh Deployment Manifest.
* **persistent_disk_type**: overwrites the Persistent Disk Type for *all* VMs in *all* Instance Groups of the Bosh Deployment Manifest.
* **networks**: overwrites the networks in *all* VMs in *all* Instance Groups of the Bosh Deployment Manifest. It currently supports multi network overwrites (and also floating networks with VIPs). The only important thing here is: the name and number of networks in the Metadata definition need to match the networks in the Bosh Deployment Manifest within the Service Broker. 

**Instance Group Specific Configuration of VMs**
```yaml
instance_group_config:
- name: haproxy
  nodes: 1
  vm_type: minimal
  persistent_disk_type: 10GB
  networks:
    - default:
      - dns
      - gateway
      name: service
    - name: floating
  properties:
    ha_proxy:
      stats_enable: false
      stats_password: me
      stats_user: overwrite
``` 
* **nodes**: overwrites the number of instances for a specific Instance Group of the Bosh Deployment Manifest.
* **vm_type**: overwrites the VM Type for a specific Instance Group of the Bosh Deployment Manifest.
* **persistent_disk_type**: overwrites the Persistent Disk Type for a specific Instance Group of the Bosh Deployment Manifest.
* **networks**: overwrites the networks in a specific Instance Group of the Bosh Deployment Manifest. It currently supports multi network overwrites (and also floating networks with VIPs). The only important thing here is: the name and number of networks in the Metadata definition need to match the networks in the Bosh Deployment Manifest within the Service Broker.
* **properties**: overwrites values for a specific Job in an Instance Group. These configuration properties, which rely to the Instance Group and are defined in the Bosh Spec of a Job.

```yaml
schemas:
    service_binding:
        create:
        parameters:
            properties:
            admin:
                description: is an admin
                type: boolean
            connections:
                description: connections
                type: number
            database:
                description: Specify the database for the service binding.
                type: string
            list:
                description: a simple list
                type: array
            schema: http://json-schema.org/draft-04/schema#
            type: object
    service_instance:
        create:
        parameters:
            properties:
            mail:
                description: mail for reference
                pattern: ^[_a-z0-9-]+(.[a-z0-9-]+)@[a-z0-9-]+(.[a-z0-9-]+)*(.[a-z]{2,4})$
                type: string
            name:
                description: name of the db that will be created
                maxLength: 7
                minLength: 3
                type: string
            value:
                description: a value
                maximum: 10
                minimum: 2
                type: integer
            schema: http://json-schema.org/draft-04/schema#
            type: object
```

The OSB Framework offers complete support for the JSON Schema Defintion to provide additional and predefined parameters during Service Instance Creation/Update and Service Bindings.

**(Partially Optional):** Custom Dashboard Endpoints
```yaml
endpoints:
  custom:
  - identifier: osb-backup-manager
    url: https://osb-backup-manager.cf.dev.eu-de-central.msh.host
  - identifier: osb-log-metric-backend
    url: https://osb-log-metric-dashboard-backend.cf.dev.eu-de-central.msh.host
  - identifier: osb-autoscaler-core
    url: https://osb-autoscaler-core.cf.dev.eu-de-central.msh.host
  default: https://osb-couchdb.cf.dev.eu-de-central.msh.host
```

**(Optional):** Service Key Manager 
```yaml
haproxy:
  auth:
    token: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
  uri: https://osb-service-key-manager.cf.dev.eu-de-central.msh.host/agents/5b07c382ab86f100075e1292/schemas?type=listen
```
The Service Key Manager is described in detail in a later section. It enables Service Brokers to support the Service Key functionality of Cloud Foundry

**(Optional):** Site Configuration 
```yaml
site:
  properties:
    osb-lbaas:
      backend_servers:
      - 10.0.8.13
      tcp_backend_servers:
      - 10.0.8.14
```

The Site Configuration is an additional way of configuring the Service Broker with custom properties, which are Site specific. In this example we see the configuration of IPs of backend servers, which are used to configure the Service Broker Deployment Manifest during deployment time. 

# Service Keys
Service Keys is a functionality introduced by Cloud Foundry, which enables users to create credentials, which are not bound to an specific application but instead used to connect to a Servce Instance from the outside. A Service Key can be created in Cloud Foundry via `cf create-service-key serviceInstanceName`. 

Offering this functionality makes sense, when you have Services like database, where user might want to access the database with a fat client to analyse problems. To offer this functionality in your Service Broker add a class named `HaProxyServiceImpl` to: `de.evoila.cf.broker.haproxy`, with the following contents:

```java
@Service
@ConditionalOnBean(HAProxyConfiguration.class)
public class HAProxyServiceImpl extends HAProxyService {

	@Override
	public Mode getMode(ServerAddress serverAddress) {
		return Mode.TCP;
	}
	
	@Override
	public List<String> getOptions(ServerAddress serverAddress) {
		return new ArrayList<>();
	}
}
```

Having defined this class, the Service Key functionality should be autowired and configured automatically. It reuses the `bind-service` functionality of the Service Broker to create Service Keys.

*Note: to enable Service Keys via Service Broker Dashboard in UI see Dashboard section and configure `serviceKeys: true` in the module definition.*

```js
sharedModules: {
  general: true,
  backup: false,
  serviceKeys: true // This needs to be set to truw
}
```

# Backup Agent
The Backup Manager is a component provided with the evoila OSB Framework in the Enterprise Package. It enables to schedule automatic backups of any endpoint. That means,each endpoint that has a concrete implementation in the `osb-backup-manager` enables customers/developers to do automatic backups of their Service Instances. 


## Configuration in the Service Definition
To enable the Backup Connector configuration wise, the Service Broker needs to be deployed with the following settings:

```yaml
plans
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

# IDE & Runtime

# Contribution

## Create an Issue

In case you find a bug, don't understand the documentation or have a question about the project you can create an issue. If you don't know how issues work, check out the link:https://guides.github.com/features/issues/[Github issues guide].

When creating an issue you should follow these tips:

- Check if a similar issue *already exists*.
- Describe your problem *as clear as possible*. What was your expected outcome and what happened instead?
- Name your *system details*, for example what operation system or library you are using.
- Paste your *error or logs* in the issue. Make sure to wrap it in three backticks to render it automatically.

## Pull Request

If you are able to patch a bug or add a feature, you can create a pull request with the code to contribute. But first of make sure you understand the license. Once you created the pull request, the maintainer(s) can check out your code and decide whether or not to pull in your changes.

When creating a pull request you should follow these tips:

- [Fork](https://guides.github.com/activities/forking/) the *repository* and *clone* it locally. Connect your local repository to the original.
- *Pull changes* as often as possible to stay up to date, so that merge conflicts will be less likely.
- [Branch](https://guides.github.com/introduction/flow/) for your changes.
- Run your changes against *existing tests*, or create new ones. 
- Follow the *style of the project*, to make it easier for the maintainer to merge.

If you are asked to make some changes to your request, add more commits and push them to your branch. These changes will automatically go into your existing pull request.

# License
Apache License Version 2.0, January 2004 http://www.apache.org/licenses/

TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

1. Definitions.
"License" shall mean the terms and conditions for use, reproduction, and distribution as defined by Sections 1 through 9 of this document.

"Licensor" shall mean the copyright owner or entity authorized by the copyright owner that is granting the License.

"Legal Entity" shall mean the union of the acting entity and all other entities that control, are controlled by, or are under common control with that entity. For the purposes of this definition, "control" means (i) the power, direct or indirect, to cause the direction or management of such entity, whether by contract or otherwise, or (ii) ownership of fifty percent (50%) or more of the outstanding shares, or (iii) beneficial ownership of such entity.

"You" (or "Your") shall mean an individual or Legal Entity exercising permissions granted by this License.

"Source" form shall mean the preferred form for making modifications, including but not limited to software source code, documentation source, and configuration files.

"Object" form shall mean any form resulting from mechanical transformation or translation of a Source form, including but not limited to compiled object code, generated documentation, and conversions to other media types.

"Work" shall mean the work of authorship, whether in Source or Object form, made available under the License, as indicated by a copyright notice that is included in or attached to the work (an example is provided in the Appendix below).

"Derivative Works" shall mean any work, whether in Source or Object form, that is based on (or derived from) the Work and for which the editorial revisions, annotations, elaborations, or other modifications represent, as a whole, an original work of authorship. For the purposes of this License, Derivative Works shall not include works that remain separable from, or merely link (or bind by name) to the interfaces of, the Work and Derivative Works thereof.

"Contribution" shall mean any work of authorship, including the original version of the Work and any modifications or additions to that Work or Derivative Works thereof, that is intentionally submitted to Licensor for inclusion in the Work by the copyright owner or by an individual or Legal Entity authorized to submit on behalf of the copyright owner. For the purposes of this definition, "submitted" means any form of electronic, verbal, or written communication sent to the Licensor or its representatives, including but not limited to communication on electronic mailing lists, source code control systems, and issue tracking systems that are managed by, or on behalf of, the Licensor for the purpose of discussing and improving the Work, but excluding communication that is conspicuously marked or otherwise designated in writing by the copyright owner as "Not a Contribution."

"Contributor" shall mean Licensor and any individual or Legal Entity on behalf of whom a Contribution has been received by Licensor and subsequently incorporated within the Work.

2. Grant of Copyright License.
Subject to the terms and conditions of this License, each Contributor hereby grants to You a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable copyright license to reproduce, prepare Derivative Works of, publicly display, publicly perform, sublicense, and distribute the Work and such Derivative Works in Source or Object form.

3. Grant of Patent License.
Subject to the terms and conditions of this License, each Contributor hereby grants to You a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable (except as stated in this section) patent license to make, have made, use, offer to sell, sell, import, and otherwise transfer the Work, where such license applies only to those patent claims licensable by such Contributor that are necessarily infringed by their Contribution(s) alone or by combination of their Contribution(s) with the Work to which such Contribution(s) was submitted. If You institute patent litigation against any entity (including a cross-claim or counterclaim in a lawsuit) alleging that the Work or a Contribution incorporated within the Work constitutes direct or contributory patent infringement, then any patent licenses granted to You under this License for that Work shall terminate as of the date such litigation is filed.

4. Redistribution.
You may reproduce and distribute copies of the Work or Derivative Works thereof in any medium, with or without modifications, and in Source or Object form, provided that You meet the following conditions:

(a) You must give any other recipients of the Work or Derivative Works a copy of this License; and

(b) You must cause any modified files to carry prominent notices stating that You changed the files; and

(c) You must retain, in the Source form of any Derivative Works that You distribute, all copyright, patent, trademark, and attribution notices from the Source form of the Work, excluding those notices that do not pertain to any part of the Derivative Works; and

(d) If the Work includes a "NOTICE" text file as part of its distribution, then any Derivative Works that You distribute must include a readable copy of the attribution notices contained within such NOTICE file, excluding those notices that do not pertain to any part of the Derivative Works, in at least one of the following places: within a NOTICE text file distributed as part of the Derivative Works; within the Source form or documentation, if provided along with the Derivative Works; or, within a display generated by the Derivative Works, if and wherever such third-party notices normally appear. The contents of the NOTICE file are for informational purposes only and do not modify the License. You may add Your own attribution notices within Derivative Works that You distribute, alongside or as an addendum to the NOTICE text from the Work, provided that such additional attribution notices cannot be construed as modifying the License.

You may add Your own copyright statement to Your modifications and may provide additional or different license terms and conditions for use, reproduction, or distribution of Your modifications, or for any such Derivative Works as a whole, provided Your use, reproduction, and distribution of the Work otherwise complies with the conditions stated in this License.

5. Submission of Contributions.
Unless You explicitly state otherwise, any Contribution intentionally submitted for inclusion in the Work by You to the Licensor shall be under the terms and conditions of this License, without any additional terms or conditions. Notwithstanding the above, nothing herein shall supersede or modify the terms of any separate license agreement you may have executed with Licensor regarding such Contributions.

6. Trademarks.
This License does not grant permission to use the trade names, trademarks, service marks, or product names of the Licensor, except as required for reasonable and customary use in describing the origin of the Work and reproducing the content of the NOTICE file.

7. Disclaimer of Warranty.
Unless required by applicable law or agreed to in writing, Licensor provides the Work (and each Contributor provides its Contributions) on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied, including, without limitation, any warranties or conditions of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A PARTICULAR PURPOSE. You are solely responsible for determining the appropriateness of using or redistributing the Work and assume any risks associated with Your exercise of permissions under this License.

8. Limitation of Liability.
In no event and under no legal theory, whether in tort (including negligence), contract, or otherwise, unless required by applicable law (such as deliberate and grossly negligent acts) or agreed to in writing, shall any Contributor be liable to You for damages, including any direct, indirect, special, incidental, or consequential damages of any character arising as a result of this License or out of the use or inability to use the Work (including but not limited to damages for loss of goodwill, work stoppage, computer failure or malfunction, or any and all other commercial damages or losses), even if such Contributor has been advised of the possibility of such damages.

9. Accepting Warranty or Additional Liability.
While redistributing the Work or Derivative Works thereof, You may choose to offer, and charge a fee for, acceptance of support, warranty, indemnity, or other liability obligations and/or rights consistent with this License. However, in accepting such obligations, You may act only on Your own behalf and on Your sole responsibility, not on behalf of any other Contributor, and only if You agree to indemnify, defend, and hold each Contributor harmless for any liability incurred by, or claims asserted against, such Contributor by reason of your accepting any such warranty or additional liability.

END OF TERMS AND CONDITIONS

APPENDIX: How to apply the Apache License to your work.

To apply the Apache License to your work, attach the following boilerplate notice, with the fields enclosed by brackets "{}" replaced with your own identifying information. (Donâ€™t include the brackets!) The text should be enclosed in the appropriate comment syntax for the file format. We also recommend that a file or class name and description of purpose be included on the same "printed page" as the copyright notice for easier identification within third-party archives.

Copyright 2018 evoila GmbH

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.