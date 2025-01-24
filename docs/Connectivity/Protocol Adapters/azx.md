---
tags:
  - connector
---
# Azure IoTHub Connector

## External Resources
- PTC Help Center - [ThingWorx Azure IoT Hub Connector](https://support.ptc.com/help/thingworx/azure_connector_scm/en/){:target="\_blank"}
- PTC Tutorial - [Connect the Azure MXChip Development Kit to Thingworx](https://community.ptc.com/t5/IoT-Tips/Azure-MXChip-Development-Kit-Learning-Path/ta-p/838928){:target="\_blank"}

## Overview

Protocol adapter for integrating Azure IoT Hub and Azure Blob Storage with Thingworx.
The connector's nickname is **AZX**.

### Key Capabilities

- Ingest **device-to-cloud** (telemetry) data from Azure (Edge) Devices
- Send **cloud-to-device** (c2d) data and invoke **Direct Methods** on Azure (Edge) Devices
- Manage **File Transfers** with Azure Blob Storage containers
- Azure Industrial IoT (IIoT) OPC UA integration (not covered in this document)

## Entities

See also [Class Diagram](#class-diagram){ data-preview }

*   The **Thingworx Azure IoT Connector** is a standalone Java executable. 
    
    It is exposed on the platform as an `AzureIotHubTemplate` Thing. That entity holds the connector configurations and handles the interactions with it.

*   An **Azure (edge) Device**  is defined in Thingworx as an `AzureIotThing` Thing. It is linked to the `AzureIotHubTemplate` Thing via the `gatewayThing` property.

    Ingress and Egress are exposed as user-defined Remote Properties and Services on the `AzureIotThing` Things.

	  The routing of IoT Hub messages to and from Thingworx relies on the **"same name"** convention:

    | Platform                   |  | Azure IotHub                         |
    | --------------------------| :-: | ------------------------------------ |
    | Thing name               |=| `deviceId` |
    | Remote Property name [(Ingress)](#ingress) |=| Device-to-cloud (telemetry) message data name  |
    | Remote Property name [(Egress)](#cloud-to-device-message) |=| Cloud-to-device (c2d) message data name |
    | Remote Service name [(Egress)](#direct-method) |=| Direct method name |

    For IoT Edge Devices, the remote services or properties must be defined using the format: `moduleID`::`methodName` or`moduleID`::`propertyName`.

*   An **Azure Storage Account** is exposed in Thingworx as an `AzureBlobStorageTemplate` Thing. It is possible to register up to 2 Azure Storage Accounts. One (required) is used internally by the connector as a checkpoint store (eventProcessorHostBlobThing), and one (optional) for File storage (fileRepositoryBlobThing). It is not recommended to use the same Storage Account for both use cases.

    A blob storage **container** used for File storage is defined in Thingworx as `AzureStorageContainerFileRepository` Things. It implements `FileRepository` and behave as such.

*   The `ConnectionServicesHub` Thing holds helper services, primarily for device provisioning and job activities.

## Features

### Device Provisioning

The `ConnectionServicesHub` holds services and events related to device provisioning and lifecycle.

#### Services
* `CreateAzureIotDevice` — Registers a new device in your Azure IoT Hub
* `DeleteAzureIotDevice` — Removes the specified device from your Azure IoT Hub
* `ImportAzureIotDevices` — Imports devices from an Azure IoT Hub into the platform as Things. The `AzureThingImportStatusEvent` is fired from the `ConnectionServicesHub` when the import has completed.

#### Events
* `AzureThingLifecycleEvent` — This event is fired after a lifecycle event occurs on a device in Azure IoT Hub. The event fields are:
    * `deviceId`: Azure IoT device identifier
    * `status`: Lifecycle status, such as deleteDeviceIdentity or createDeviceIdentity
    * `twin`: Value of the device twin at the time of creation or deletion.
The Azure IoT Hub must be configured to route the Device Lifecycle Events to the endpoint consumed by the Connector (generally be enabled by default).

### Ingress
Device-to-cloud (telemetry) messages are exposed on the `AzureIotThing` Thing as **Remote Properties**.

##### Examples 1
```json
{"messageId":0,"deviceId":"Devkit playground","temperature":"24.39","humidity":"77.84"}
```
The **temperature** reading can be exposed as a remote property named `temperature` of type NUMBER.

##### Examples 2
```json
{"messageId":0,"deviceId":"Devkit playground","payload": {"temperature":"24.39","humidity":"77.84"}}
```
The **temperature** reading is nested and cannot directly be mapped to a Thingworx property. The **payload** can be exposed as a remote property named `payload` of type JSON. The temperature can subsequently be extracted from the payload property, for example from a DataChange subscription.

### Egress

#### Cloud-to-device message
cloud-to-device (c2d) messages are exposed on the `AzureIotThing` Thing as **Remote Properties**.

#### Direct Method
Direct Methods (Device Methods) are exposed on the `AzureIotThing` Thing as **Remote Services**.

### File Transfers
An Azure Blob Storage Container exposed in Thingworx as `AzureStorageContainerFileRepository` behaves like a normal (remote) **FileRepository**.

### Device jobs

## Quick Setup (Windows)

!!! warning "For testing only"

1. Pre-requisites
    * an Azure IoTHub
    * an Azure Storage Account with specific requirements:
	    * Firewall is not supported [:material-link:](https://www.ptc.com/en/support/article/CS350044){:target="\_blank"}
	    * Hierarchical namespace, Blob soft delete, Versioning must be disabled on the Storage used for checkpointing  [:material-link:](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-java-get-started-send?tabs=passwordless%2Croles-azure-portal#receive-events){:target="\_blank"}
    * a Thingworx platform
2. Download and extract the **ThingWorx Azure IoT Hub Connector** archive from [eSupport portal](https://support.ptc.com/appserver/auth/it/esd/product.jsp?prodFamily=TWC){:target="\_blank"}
3. On the platform:
    * Create an `Administrator` appKey
    * Import the 2 extensions from `<AZX>\extensions` (ConnectionService first)
    * Create a Thing that implements the `AzureBlobStorageTemplate` Template and [configure](https://support.ptc.com/help/thingworx/azure_connector_scm/en/#page/thingworx_scm_azure%2Fazure_connector%2Fc_azure_connector_create_azure_entities_in_thingworx.html){:target="\_blank"} it for your Azure Storage Account
    * Create a Thing that implements the `AzureIotHubTemplate` Template and [configure](https://support.ptc.com/help/thingworx/azure_connector_scm/en/#page/thingworx_scm_azure%2Fazure_connector%2Fc_azure_connector_create_azure_entities_in_thingworx.html){:target="\_blank"} it for your Azure IoTHub
4. On the connector:
    * Edit `<AZX>\connector\bin\azure-iot.bat` and add the following line after `set DEFAULT_JVM_OPTS=...`:
    ```bat
    set AZURE_IOT_OPTS=-Dconfig.file=%APP_HOME%\conf\azure-iot.conf -Dlogback.configurationFile=%APP_HOME%\conf\logback.xml -Dconfig.plaintext=true
    ```
    * Edit `<AZX>\connector\conf\azure-iot.conf` and update the following settings:
        * `cx-server.transport.websockets.app-key`: the appKey 
        * `cx-server.transport.websockets.platforms`: the Thingworx WS endpoint URL
        * `cx-server.protocol.hub-thing-name`: the name of the `AzureIotHubTemplate` Thing created previously
    * Start the Connector with `<AZX>\connector\bin\azure-iot.bat`
5. Testing:
    * Use the [MXChip IoT DevKit Simulator](https://azure-samples.github.io/iot-devkit-web-simulator/){:target="\_blank"} > Get Started as Azure Device
    * From the [Azure portal](https://portal.azure.com){:target="\_blank"}, create an Azure Device named **MXCHIP_SIM**   (or use the `CreateAzureIotDevice` service on `ConnectionServicesHub` from Thingworx)
	    * Get the device connection string from the Azure portal and set it on the simulator
	    * Run the Simulator. The simulator telemetry messages have the following format:
		    ```json
		    {"messageId":0,"deviceId":"Devkit playground","temperature":"23.72","humidity":"71.35"}
		    ```
    * From Composer, create an `AzureIotThing` Thing named **MXCHIP_SIM** (or use the `ImportAzureIotDevices` service on  `ConnectionServicesHub`)
	    * Update the `gatewayThing` property to reference the `AzureIotHubTemplate` Thing
	    * Create remote properties named `temperature` and `humidity` with type NUMBER
	* Verify that the temperature and humidity readings are updated on the **MXCHIP_SIM** Thing
  
See also [Azure MXChip Development Kit Learning Path](https://community.ptc.com/t5/IoT-Tips/Azure-MXChip-Development-Kit-Learning-Path/ta-p/838928)

## Troubleshooting

## Class Diagram

``` mermaid
---
  config:
    class:
      hideEmptyMembersBox: true
---
classDiagram
  AzureBlobStorageTemplate .. AzureStorageContainerFileRepository:fileRepositoryBlobThing@AzureIotHubTemplate
  AzureIotHubAdapterServices <|-- ConnectionServicesHub
  AzureIotThing <|-- myAzureDevice
  AzureIotHubTemplate <|-- myAzureIotHub
  AzureStorageContainerFileRepository <|-- myAzureFileRepository
  AzureIotHubTemplate <-- AzureIotThing:gatewayThing
  myAzureIotHub <-- myAzureDevice:gatewayThing
  AzureBlobStorageTemplate <-- AzureIotHubTemplate:fileRepositoryBlobThing
  AzureBlobStorageTemplate <-- AzureIotHubTemplate:eventProcessorHostBlobThing
  style myAzureIotHub fill:#ffffc5
  style myAzureDevice fill:#ffffc5
  style myAzureFileRepository fill:#ffffc5
  classDef thing fill:#ffffc5
  class AzureIotHubTemplate{
    <<Template>>
    Azure IoTHub configs : Configuration
  }
  class myAzureIotHub:::thing{
    <<Thing>>
  }
  class AzureIotHubAdapterServices{
    <<Shape>>
    CreateAzureIotDevice()
    DeleteAzureIotDevice()
    DescribeAzureIotDevice()
    ImportAzureIotDevices()
  }
  class ConnectionServicesHub{
    <<Thing>>
  }
  class AzureIotThing{
    <<Template::RemoteThing>>
    twinDesired : JSON
    twinReported : JSON
    twinTags : JSON
  }
  class myAzureDevice:::thing{
    <<Thing>>
    myTelemetryData : RemoteProperty
    myc2dData : RemoteProperty
    myDeviceMethod() : RemoteService 
  }
  class AzureBlobStorageTemplate{
    <<Template>>
    accountName : Configuration
    connectionString : Configuration
  }
  class AzureStorageContainerFileRepository{
    <<Template::FileRepository>>
    containerName : Configuration
  }
  class myAzureFileRepository:::thing{
    <<Thing>>
  }
```