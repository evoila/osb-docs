# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](service-keys.md)
5. [Backup Agent](backup-agent.md)
6. [Development](development.md)
7. [Open Service Broker API Features](osb-api-features.md)
    * [7.1 Open Service Broker API v2.13](#open-service-broker-api-v2.13)
    * [7.2 Open Service Broker API v2.14](#open-service-broker-api-v2.14)
    * [7.3 Open Service Broker API v2.15](#open-service-broker-api-v2.15)
8. [IDE & Runtime](ide-runtime.md)
9. [Contribution](contribution.md)
10. [License](license.md)

# Implemented API Features

This Documents sums up all OSB-API features, that got added after v2.12, the frameworks implements so far. The framework supports all API features up to version 2.14 and is currently being upgraded to support 2.15. This is not a detailed documentation of these features, but a overview for developers. Please refer to OSB-API Docs for more information.

## Open Service Broker API v2.13

- Schemas: Schema definitions for service instances and bindings for the plan. Used to declare configuration parameters for creating a service instance, updating a service instance and creating a service binding.

- Context field: Platform specific contextual information under which the service instance is to be provisioned. Will replace organization_guid and space_guid in future versions of the specification.

- Originating Identity: Often a service broker will need to know the identity of the user that initiated the request from the platform. For example, this might be needed for auditing or authorization purposes. In order to facilitate this, the platform will need to provide this identification information to the broker on each request. 

- Service Metadata: a field to store metadata objects in service definitons and plans.

## Open Service Broker API v2.14

- Fetching: Added GET endpoints for service instances and bindings.

- Asynchronous Bindings: Bindings can be performed asynchronously and a polling endpoint has been added.

## Open Service Broker API v2.15

- Request Identity: A platform might wish to uniquely identify a specific request. This might be used for logging and request tracing purposes.

- Maintenance Info: The service catalogs now holds an object to specify consequences of an provision or update action. This gives the user information about the steps initiated by the service broker when triggering such an action.

<p align="center">
    <span ><a href="development.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="ide-runtime.md">Next -></a></span>
</p>
