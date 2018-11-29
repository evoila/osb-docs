# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](#components)
    * 2.1 [Core](#core)
        * 2.1.1 [Core](#core)
        * 2.1.2 [Model](#model)
        * 2.1.3 [Security](#security)
    * 2.2 [Persistence](#persistence)
    * 2.3 [Deployment](#deployment)
        * 2.3.1 [General](#general)
            * 2.3.1.1 [Create Instance](#create-instance)
            * 2.3.1.2 [Update Instance](#update-instance)
            * 2.3.1.3 [Delete Instance](#delete-instance)
        * 2.3.2 [Existing-Service](#existing-service)
        * 2.3.3 [Bosh](#bosh)
            * 2.3.3.1 [Invocation of Bosh functionality](#invocation-of-bosh-functionality)
        * 2.3.4 [Openstack (Deprecated)](#openstack-deprecated)
    * 2.4 [Dashboard](#dashboard)
    * 2.5 [Credhub](#credhub)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](service-keys.md)
5. [Backup Agent](backup-agent.md)
6. [Development](development.md)
7. [IDE & Runtime](ide-runtime.md)
8. [Contribution](contribution.md)
9. [License](license.md)
---

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

## Credhub

With the credhub integration it is now possible to generate, store and use credentials or certificates on the fly.

### Requirements

Before using the feature make sure to save all necessary information in the **application.yml**. These **MUST** be saved as follows:

    spring:
        credhub:
            url: https://<bosh-director>:8844
            bosh-director: <bosh-director-name>
            keystore-password: <keystore-password>
            oauth2:
                client-id: <client-id>
                client-secret: <client-secret>
                access-token-uri: https://<bosh-director>:8443/oauth/token
            certificate:
                ca: <ca-cert>
                cert: <client-cert>
                private-key: <private-key>

#### Description

| Field                                  | Description   |
| -------------------------------------- | ----------------- |
| spring.credhub.url                     | URL of the environment where credhub is installed |
| spring.credhub.bosh-director           | Name of the bosh director |
| spring.credhub.keystore-password       | **(OPTIONAL)** User generated password to secure the keystore |
| spring.credhub.oauth2.client-id        | Username for fetching an oauth token |
| spring.credhub.oauth2.client-secret    | Password for fetching an oauth token |
| spring.credhub.oauth2.access-token-uri | Uri to fetch an oauth token from, which is used to authenticate for generating, storing & deleting credentials |
| spring.credhub.certificate.ca          | Credhub CA certificate, found in the creds.yml of the bosh director under "**credhub_ca.ca**" |
| spring.credhub.certificate.cert        | Credhub client certificate, found in the creds.yml of the bosh director under "**credhub_ca.certificate**" |
| spring.credhub.certificate.private-key | Credhub private key, found in the creds.yml of the bosh director under "**credhub_ca.private_key**"|

---

### Generating

The credhub client can either generate an user (consisting of usernamen and password), a standalone password, a json file or a certificate.

```java

public void createUser(String instanceId, String valueName, String username, int passwordLength) { ... }

public void createPassword(String instanceId, String valueName, int passwordLength) { ... }

public void createJson(String intanceId, String valueName, Map<String, Object> values) { ... }

public void createCertificate(String instanceId, String valueName, CertificateParameters certificateParameters) { ... }
```

Credentials are stored as `/<bosh-director-name>/sb-<instanceId>/<valueName>`, for example `/bosh-1/sb-dc4ac700-ee62-4b24-817b-f1b10f2d2d40/redisPassword`.

#### Description

| Parameter                                  | Description   |
| -------------------------------------- |-----------------|
| instanceId | ID of the service instance|
| username | Name of the user (for the user entry only) |
| valueName | Name of the value you want to save|
| passwordLength| **(OPTIONAL)** Length of the password, default is 40 |
| values | A map of values that should be saved as json |
| certificateParameters | Parameters for a certificate, created by a CertificateParametersBuilder. See [here](https://docs.spring.io/spring-credhub/docs/1.0.0.RELEASE/api/org/springframework/credhub/support/certificate/CertificateParameters.CertificateParametersBuilder.html)|

### Usage

#### Manifest

The manifest.yml can access credhub credentials as follows

##### User

    properties:
        ...:
            username: ((<valueName>.username))
            password: ((<valueName>.password))

##### Password

    properties:
        ...:
            password: ((<valueName>))

##### JSON

    properties:
        ...:
            anyField: ((<valueName>.<anyValueFromJson>))

##### Certificates

    properties:
        ...:
            certificate: ((<valueName>.certificate))
            ca: ((<valueName>.ca))
            private_key: ((<valueName>.private_key))

#### Java

You can also access credhub credentials in java with the following methods:

```java

//User credentials

public String getUserName(String instanceId, String valueName) { ... }

public String getUserPassword(String instanceId, String valueName) { ... }

//Password credentials

public String getPassword(String instanceId, String valueName) { ... }

//JSON credentials

public Object getJson(String instanceId, String valueName, String key) { ... }

//Certificate credentials

public String getCertificate(String instanceId, String valueName) { ... }

public String getCertificateAuthority(String instanceId, String valueName) { ... }

public String getPrivateKey(String instanceId, String valueName) { ... }

```

Parameters are the same as in [Generating](#generating)

---

<p align="center">
    <span ><a href="../README.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="configure-service-broker.md">Next -></a></span>
</p>
