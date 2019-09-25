# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](#configuring-a-service-broker)
    * 3.1 [Basic Configuration](#basic-configuration)
    * 3.2 [Catalog Configuration](#catalog-configuration)
      * 3.2.1 [Dashboard Client](#dashboard-client)
      * 3.2.2 [Plans](#plans)
        * 3.2.2.1 [Platform](#platform)
        * 3.2.2.2 [Metadata](#metadata)
4. [Service Keys](service-keys.md)
5. [Backup Agent](backup-agent.md)
6. [Development](development.md)
7. [Open Service Broker API Features](osb-api-features.md)
8. [IDE & Runtime](ide-runtime.md)
9. [Contribution](contribution.md)
10. [License](license.md)

---


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

**Java Version**

The configuration of the service broker and the deployment manifest depends on the Java version you are working with. For version 9 or higher you have to add the following compiler arguments to **maven-compiler-plugin** in your **pom.xml** since the sun.* packages are no longer a part of the Java interface.

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<configuration>
		<source>${java.version}</source>
		<target>${java.version}</target>
		<compilerArgs>
			<arg>--add-exports</arg>
			<arg>java.base/sun.security.util=ALL-UNNAMED</arg>
		</compilerArgs>
	</configuration>
</plugin>
```

Also make sure to change to Java version in the **pom.xml**

```xml
<java.version>12</java.version>
```

Change the spring-boot-starter-parent version in the **pom.xml** as well

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.1.7.RELEASE</version>
</parent>
```

For deploying the service-broker on Cloud Foundry you also have to change the deployment **manifest.yml**. First you have to specify the buildpack version. For Java 12 this would be

```yaml
buildpack: https://github.com/cloudfoundry/java-buildpack.git#v4.19.1
```

Furthermore if the spring cloud config is used, the spring-cloud.version has to be updated to the Greenwich release.

```yaml
<spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
```

Last but not least add the following line to the environment section in the manifest and you are good to go.

```yaml
env:
  JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 12.+ } }'
```

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

**(Optional):** Enabling Service Keys
To enable the handling and creation of Service Keys in the Service Broker, add and set the following property to `true` in your configuration.
```yaml
service-keys:
    enabled: true
```

*Note: To enable Service Keys via Service Broker Dashboard in UI see Dashboard section and configure `serviceKeys: true` in the module definition.*

```js
sharedModules: {
  general: true,
  backup: false,
  serviceKeys: true // This needs to be set to true
}
```

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

---

<p align="center">
    <span ><a href="components.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="service-keys.md">Next -></a></span>
</p>
