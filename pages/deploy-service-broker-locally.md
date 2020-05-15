# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](service-keys.md)
5. [Backup Agent](backup-agent.md)
6. [Development](development.md)
7. [Local Deployment](#how-to-depoly-a-service-broker-on-your-local-machine-with-bosh-lite)
    * 7.1 [Prerequisite](#prerequisite)
    * 7.2 [Bosh-lite config and service broker connection](#bosh-lite-config-and-service-broker-connection)
      * 7.2.1 [Connect the service-broker to bosh-lite](#connect-the-service-broker-to-bosh-lite)
      * 7.2.2 [Prepare bosh and service broker manifest](#prepare-bosh-and-service-broker-manifest)
    * 7.3 [Setup Security Components](#setup-security-components)
      * 7.3.1 [Bosh-lite Uaa setup](#bosh-lite-uaa-setup)
      * 7.3.2 [Connect to bosh-lite credhub](#connect-to-bosh-lite-credhub)
    * 7.3 [Creds.yml fields](#creds.yml-fields)
8. [Open Service Broker API Features](osb-api-features.md)
9. [IDE & Runtime](ide-runtime.md)
10. [Contribution](contribution.md)
11. [License](license.md)
---

# How to depoly a service broker on your local machine with bosh-lite
This tutorial describes how to set up a service broker locally with bosh-lite. It is focused on deploying a already created project based on osb-core.
After following these steps you should be able to use your service broker to deploy a service on bosh-lite, manage it with the service-broker dashboard, save credentials in Credhub and use Uaa to authenticate users. Uaa and credhub are being deployed as a part of Bosh-lite.

## Prerequisite
Before deploying your service broker you need a running bosh-lite, a mongoDB instance and the bosh release you wish to deploy. Note your service broker may needs to implement some feature to handle deployments correctly. If you don't have deployed bosh-lite yet, refer to the bosh documentation for instructions. For a mongoDB deployment a quick solution is docker. Read [here](https://phoenixnap.com/kb/docker-mongodb) on how to host mongoDB on docker.

## Bosh-lite config and service broker connection
This subsection explains all necessary configuration for deploying a service on bosh-lite with a service broker.

### Connect the service-broker to bosh-lite
To use your service broker to deploy services on bosh-lite you need configure it in your application.yml file, like this:
```yaml
bosh:
  host: 192.168.50.6
  username: admin
  password: ((the bosh admin secret))
  stemcellOs: ((the Stemcell you are using))
  stemcellVersion: latest
  vip_network: floating
```

If not modified by you, the `host` and `username` are always `192.168.50.6` and `admin`. The password can be found in the by bosh-lite automatically generated `creds.yml` file under `admin_password` To find the value for `stemcellOs` run the command `bosh env`. It will return something like this:
```
Name               bosh-lite  
UUID               857d2ab1-c78b-4d93-9177-699267e94caf  
Version            270.1.0 (00000000)  
Director Stemcell  ubuntu-xenial/315.34  
CPI                warden_cpi  
Features           compiled_package_cache: disabled  
                   config_server: enabled  
                   local_dns: enabled  
                   power_dns: disabled  
                   snapshots: disabled  
User               admin 
```
You can find under `Director Stemcell` the needed value and the `stemcellVersion`.

## Prepare bosh and service broker manifest
The service you wish to deploy needs to be released on bosh-lite. To do this, create and upload your bosh release according to the [bosh documentation](https://bosh.io/docs/release/).

Your service broker needs a `manifest.yml` file located under `resources/bosh`, for the bosh-release you wish to deploy. You need to ensure that this manifest is configured according to the bosh cloud configuration. To read the bosh cloud-config run the command `bosh cloud-config`. Read [here](https://bosh.io/docs/sample-manifest/) to learn more about bosh release manifests.

## Setup Security Components
To access the service broker dashboard an oauth2 identity provider is necessary. As Uaa is deployed with bosh-lite anyway, it is possible to configure the service broker to use it. To configure Uaa itself, please make sure you have installed [Uaac](https://docs.pivotal.io/platform/application-service/2-8/uaa/uaa-user-management.html).

Additionally, the bosh-lite credhub can be used to store credentials, instead of mongoDB.

### Bosh-lite Uaa setup
Uaa needs an additional client and for testing we should add a user. For this we need to authenticate uaac with uaa. The necessary credentials can be found inside the `creds.yml`. To do so log into uaa with uaac, by running the following commands:
```
uaac target https://192.168.50.6:8443
uaac token client get uaa_admin -s ((uaa_admin_client_secret))
```
The `uaa_admin_client_secret` can be found in `creds.yml`. Create a new client and a user like this:
```
uaac client add webappclient -s webappclientsecret \ 
--name WebAppClient \ 
--scope resource.read,resource.write,openid,profile,email,address,phone \ 
--authorized_grant_types authorization_code,refresh_token,client_credentials,password \ 
--authorities uaa.resource \ 
--redirect_uri http://localhost:8080/custom/v2/authentication/

uaac user add appuser -p appusersecret --emails appuser@acme.com
	
uaac group add resource.read
uaac group add resource.write

uaac member add resource.read appuser
uaac member add resource.write appuser

```
For more information on uaac read [here](https://docs.pivotal.io/platform/application-service/2-8/uaa/uaa-user-management.html).

The bosh lite uaa uses the same IP as the director, and the port `8443`. The service broker configuration YAML needs to be configured like this:

```yaml
spring:
  security:
      oauth2:
        resourceserver:
          jwt:
            issuer-uri: "https://192.168.50.6:8443/oauth/token"
catalog:
  services:
      dashboard:
        auth_endpoint: "https://192.168.50.6:8443/oauth"
        url: "http://localhost:8080/custom/v2/authentication"
      dashboard_client:
        id: webappclient
        redirect_uri: "http://localhost:8080/custom/v2/authentication/"
        secret: webappclientsecret
```

To access UAA the service broker needs to trust the SSL certificate that protects Uaa. There are two options, trust all self-signed certificates or add the uaa ca from the `creds.yml` to the service broker. To trust all certificates configure this:

```yaml
spring:
  ssl:
    acceptselfsigned: true
```

Add the director ssl ca from the `creds.yml` like below, if you don't want to just accept everything:

```yaml
spring:
  ssl:
    certificates:
      uaa_ssl: |
        -----BEGIN CERTIFICATE-----
        .....
        -----END CERTIFICATE-----
```
The ca can be found in `creds.yml` under the key `director_ssl.ca`. After this the dashboard of a deployed service is accessible through its dashboard URL. The user `appuser` with password `appusersecret` can be used to log in.

### Connect to bosh-lite credhub
To use the bosh lite credhub as a credentials store, the service broker has to authenticate itself with the uaa. Credhub runs on the same IP as the director and uses the port `8844` The necessary `client-id` is called `credhub-admin` the `client-secret` is saved in the `creds.yml` under `credhub_admin_client_secret`. Credhub uses a different certificate that then uaa, so it all self-signed certificates need to be accepted or the `credhub ca` needs to be accepted in addition to the `director ca`. The `credhub ca` is saved in the `creds.yml` file in `credhub_ca.ca`. A credhub configuration with bosh lite looks like this:

```yaml
spring:
  profiles: local, credhub
  credhub:
    bosh-director: bosh-lite
    oauth2:
      registration-id: credhub-client
    url: "https://192.168.50.6:8844"

  security:
    oauth2:
      client:
        registration:
          credhub-client:
            provider: uaa
            client-id: credhub-admin
            client-secret: ((credhub_admin_secret))
            authorization-grant-type: client_credentials
        provider:
          uaa:
            token-uri: "https://192.168.50.6:8443/oauth/token"
  ssl:
    certificates:
      uaa_ssl: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      credhub_ca: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
```

## Creds.yml fields
A table with the fields in the service broker config and their matches needed from the `creds.yml` file and fix values:

| Field in service broker config | Field in creds.yml or Fix value |
| ------------------------------ | ------------------------------- |
| bosh.host                      | 192.168.50.6                    |
| bosh.username                  | admin                           |
| bosh.password                  | ((admin_password))              |
| bosh.vip_network               | floating                        |
| spring.security.oauth2.resourceserver.jwt.issuer-uri | "https://192.168.50.6:8443/oauth/token" |
| spring.security.oauth2.client.registration.credhub-client.client-id | credhub-admin |  
| spring.security.oauth2.client.registration.credhub-client.client-secret | ((credhub_admin_secret)) |
| spring.security.oauth2.client.provider.uaa.token-url | "https://192.168.50.6:8443/oauth/token" |
| spring.ssl.certificates.uaa_ssl | ((director_ssl.ca)) | 
| spring.ssl.certificates.uaa_ssl | ((credhub_ca.ca)) |

<p align="center">
    <span ><a href="development.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="osb-api-features.md">Next -></a></span>
</p>
