# Table of Contents

1. [Open Service Broker Framework (by evoila)](../README.md)
2. [Components](components.md)
3. [Configuring a Service Broker](configure-service-broker.md)
4. [Service Keys](#service-keys)
5. [Backup Agent](backup-agent.md)
6. [Development](development.md)
7. [IDE & Runtime](ide-runtime.md)
8. [Contribution](contribution.md)
9. [License](license.md)
---

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

---

<p align="center">
    <span ><a href="configure-service-broker.md"><- Previous</a></span>
	    <span>&nbsp; | &nbsp;</span> 
    <span><a href="backup-agent.md">Next -></a></span>
</p>