
### Implemented API Features

This Documents sums up all OSB-API features, that got added after v2.12, the frameworks implements so far. The framework supports all API features up to version 2.14 and is currently upgraded to support 2.15. This is not a detailed documentation of these features, but a overview for developers. Please refer to OSB-API Docs for more information.

## Open Service Broker API v2.13

- Schemas: Schema definitions for service instances and bindings for the plan. Used to declare configuration parameters for creating a service instance, updating a service instance and creating a service binding.

- Context field: Platform specific contextual information under which the service instance is to be provisioned. Will replace organization_guid and space_guid in future versions of the specification.

- Originating Identity: Often a service broker will need to know the identity of the user that initiated the request from the platform. For example, this might be needed for auditing or authorization purposes. In order to facilitate this, the platform will need to provide this identification information to the broker on each request. 

- Service Metadata: a field to store metadata objects in service definitons and plans.

## Open Service Broker API v2.14

- fetching: Added GET endpoints for service instances and bindings.

- Asynchronous Bindings: Bindings can be performed async, and a polling endpoint has been added.

- 


## Open Service Broker API v2.15

- Request Identity: A Platform might wish to uniquely identify a specific request as it flows throughout the system. For example, this might be used for logging for request tracing purposes.
