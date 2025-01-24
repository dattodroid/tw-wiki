# Data Ordering

To ensure high throughput, Thingworx heavily relies on parallel and distributed processing.
However, high concurrency can lead to requests and events being processed out of order relative to their emission time.

* Starting with Thingworx 9.5, it is possible to configure [**Subscriptions**](#sequential-subscriptions-95) to **execute sequentially**. This ensures that subscriptions are executed in the order events are triggered.
* Since 9.7, it is possible to configure Thingworx to ensure that [**Property writes**](#ordered-property-writes-97) (i.e. data pushed from devices) are **processed in order**.
* **Data Ordering** refers to the combination of the above features to ensure that data is ingested and data change events are executed in order.

!!! info
	The use of Data Ordering can impact [processing performance](https://support.ptc.com/help/thingworx/platform/r9.7/en/#page/ThingWorx/Help/ModelandDataBestPractices/DataOrderingPerformanceResults.html#){:target="\_blank"}

### Sequential Subscriptions (9.5+)

* Configurable at **Subscription** level
	* Driven by the **Execute Events Sequentially** option on the Subscription
	
### Ordered Property writes (9.7+)

* Applies to data pushed from AlwaysOn devices and Axeda agents using the `UpdateSubscribedPropertyValues` and  `UpdateSubscribedPropertyValuesBatched` services
	* CSDK - `twApi_PushSubscribedProperties()`
	* NSDK/JSDK - `VirtualThing::updateSubscribedProperties()`
	* EMS/LSR - `thingworx.server.setProperties()`
	* Axeda - Data item
* **System wide** configuration to be applied on multiple Thingworx components
	* **Platform**

		In `platform-settings.json`:
		```json
		{  
		"PlatformSettingsConfig": {  
			"BasicSettings": {  
			…  
			"EnableDataOrdering": true,
		```

	* **Connection Server (9.3+) / eMessage Connector (2.5+)**
	
		* In _connector.conf_ add `cx-server.data-ordering true`
		* It is recommended to configure the Load Balancer in front of multiple connectors to use device/ip stickiness (for both CXS and EMC)

## External Resources

* PTC Help Center - [Data Ordering](https://support.ptc.com/help/thingworx/platform/r9.7/en/#page/ThingWorx/Help/ModelandDataBestPractices/DataOrdering.html){:target="\_blank"}