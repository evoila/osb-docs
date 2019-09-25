# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](#service-keys)
5. [Backup Agent](backup-agent.md)
6. [Development](development.md)
7. [Open Service Broker API Features](osb-api-features.md)
8. [IDE & Runtime](ide-runtime.md)
9. [Contribution](contribution.md)
10. [License](license.md)
---

# Service Keys
Service Keys is a functionality introduced by Cloud Foundry, which enables users to create credentials, which are not bound to an specific application but instead used to connect to a Servce Instance from the outside. A Service Key can be created in Cloud Foundry via `cf create-service-key serviceInstanceName`. 

Offering this functionality makes sense, when you have Services like database, where user might want to access the database with a fat client to analyse problems. To offer this functionality in your Service Broker see the according section in the configuration pages of this documentation.

Having defined this property, the Service Key functionality should be autowired and configured automatically. It reuses the `bind-service` functionality of the Service Broker to create Service Keys.

Be aware, that the Service Broker does not change the underlying network architecture to realize external access to use the created Service Key, therefore you have to have access to the infrastructure, where the broker deploys its Service Instances. One way to do this is using `cf ssh` for an Application and access the Service Instance from there.

---

<p align="center">
    <span ><a href="configure-service-broker.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="backup-agent.md">Next -></a></span>
</p>
