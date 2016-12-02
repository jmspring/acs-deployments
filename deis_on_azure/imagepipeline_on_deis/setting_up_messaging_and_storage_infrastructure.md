# Setting up a Distributed Application Infrastructure

This walk through will talk about the deployment of the additional infrastructure for a
distributed application that will be built atop the infrastructure deployed in the 
[Deis on Azure](https://github.com/jmspring/acs-deployments/blob/master/deis_on_azure/deis_on_azure.md)
walkthrough.

As evident from the prior walkthrough, infrastructure components deployed in the Azure Cloud
are done so through Resource Groups.  Resource Groups are a logical grouping of resources and
can provide options for Role Based Access Control.

When deploying an application that makes use of something like Deis Workflow atop Kubernetes in
Azure, it is best to separate out the additional resources the application needs from those 
that make up the infrastructure it is running on.

In this write up, the creation and deployment of the infrastructure for the following application
outline will be deployed:

```
<Data Source> --> [Ingest] --> (Message into Processing Queue)
                     |
                     --> (Data into Blob Store)


    (Processing Queue) --> [Data Analyzer] --> (Message into Notification Queue)
                            ^      |
             (Blob Store) --|      --> (Data Analysis Output into Analysis Event Hub)


(Notification Queue) --> [Notifier]
```

The components in the diagram translate as follows:

  - **<Data Source>** represents an external device from which data is received
  - **[Ingest]**, **[Data Analyzer]**, and **[Notifier]** are services running within Deis Workflow
  - **Processing Queue** is an [Azure Service Bus Queue](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-fundamentals-hybrid-solutions#queues)
    used for notifying the the Data Analyzer of the next set of data to be processed.
  - **Notification Queue** is also an Azure Service Bus Queue.  This one is used to let Notifier know when, by analysis, the need for a notification needs to be sent.
  - **Analysys Event Hub** is an [Azure Event Hub](https://azure.microsoft.com/en-us/services/event-hubs/) to which all analysis results are sent for further processing in a later walkthrough.
  - **Blob Store** is an [Azure Blob Store](https://docs.microsoft.com/en-us/azure/storage/storage-introduction#blob-storage) in which all the data ingested will be stored for future processing.

So, for this application, we have four additional Azure infrastructure components we need to 
deploy and provision.  Those pieces, as noted above are:

  - Two Azure Service Bus Queues
  - One Azure Event Hub
  - One Azure Blob Store

It is possible to deploy these resources using the Azure Portal (https://portal.azure.com), [Azure CLI 2.0](https://github.com/Azure/azure-cli) as well as the [Azure Xplat CLI](https://github.com/Azure/azure-xplat-cli).
To stay consistent with the prior walkthroughs, the Azure CLI 2.0 will be used.  **NOTE** Due to the preview
nature of the Azure CLI 2.0, it is not possible to create all resources through the CLI.  So, the steps below
will include using existing [Azure Quickstart Templates](https://github.com/Azure/azure-quickstart-templates).
Thus the process will not be too cumbersome.

The steps for the deployments are as follows:

  - Define the set of values needed to deploy each resource
  - Create a resource group
  - Create the Azure Storage Account for storing blobs
  - Create the Azure Service Bus Queues
  - Create the Azure Event Hub
  - Retrieve Azure Service Bus and Azure Event Hub Credentials

For a tutorial of what Azure Templates are and how to work with them, please take a look [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-template-walkthrough).  

## Define the Values Needed for deployment

The application that will be deployed in the next walkthrough is a set of services for capturing and analyzing
images.  So, the names chosen below will reflect on that.

The first thing needing defining is the Resource Group.  The Resource Group will be created using the Azure CLI.  The values needed:

  - Resource Group Name:  `imagepipelinerg`
  - Location:  `West US` (for the CLI, the value is `westus`)

For the Azure Storage Account, the CLI will also be used to create the Storage Account.  There are four values needed:

  - Resource Group Name:  `imagepipelinerg` (**as above**)
  - Location:  `West US` (**as above**)
  - Storage Account Name:  `imagepipelinestore`
  - Storage Account SKU:  `Standard_LRS` (standard, locally redundant storage)

For the Azure Service Bus Queues, one of the existing [Azure Quickstart Templates](https://github.com/Azure/azure-quickstart-templates/tree/master/101-servicebus-queue).  If you examine
the [template deployment parameters](https://github.com/Azure/azure-quickstart-templates/blob/master/201-servicebus-create-queue/azuredeploy.parameters.json) the needed parameters (plus arguments for the CLI deployment command):

  - Deployment Template:  `https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-servicebus-queue/azuredeploy.json`
  - Resource Group Name:  `imagepipelinerg` (**as above**)
  - Deployment Names:
    - `imagepipelineproc-dep` (for the **Processing Queue**)
    - `imagepipelinenot-dep` (for the **Notification Queue**)
  - Service Bus Namespace:  `imagepipelinesbus`
  - Service Bus Queue Names:
    - `imagepipelineprocq` (for the **Processing Queue**)
    - `imagepipelinenotq` (for the **Notification Queue**)

For creating the Azure Event Hub, another [quickstart template](https://github.com/Azure/azure-quickstart-templates/tree/master/201-event-hubs-create-event-hub-and-consumer-group) is used.  The [parameters](https://github.com/Azure/azure-quickstart-templates/blob/master/201-event-hubs-create-event-hub-and-consumer-group/azuredeploy.parameters.json) required (plus arguments for the CLI deployment command) are:

  - Deployment Template:  `https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/201-event-hubs-create-event-hub-and-consumer-group/azuredeploy.json`
  - Resource Group Name:  `imagepipelinerg` (**as above**)
  - Deployment Name:  `imagepipelineeh-dep`
  - Event Hub Namespace:  `imagepipelineehns`
  - Event Hub Name:  `imagepipelineeh`
  - Event Hub Consumer Group:  `imagepipelineanalyze`

## Create A Resource Group

The Resource Group is created using the Azure CLI.  With the values above, the process resembles the 
following:

```bash
jims@dockeropolis:~$ az resource group create \
> --name="imagepipelinerg" \
> --location="westus"
{
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg",
  "location": "westus",
  "managedBy": null,
  "name": "imagepipelinerg",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null
}
```

## Create the Storage Account

The Storage Account is also created using the Azure CLI.  With the values from above, the process is as follows:

```bash
jims@dockeropolis:~$ az storage account create \
> --location="westus" \ 
> --name="imagepipelinestore" \
> --resource-group="imagepipelinerg" \
> --sku="Standard_LRS"
{
  "accessTier": null,
  "creationTime": "2016-12-01T21:36:25.900614+00:00",
  "customDomain": null,
  "encryption": null,
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.Storage/storageAccounts/imagepipelinestore",
  "kind": "Storage",
  "lastGeoFailoverTime": null,
  "location": "westus",
  "name": "imagepipelinestore",
  "primaryEndpoints": {
    "blob": "https://imagepipelinestore.blob.core.windows.net/",
    "file": "https://imagepipelinestore.file.core.windows.net/",
    "queue": "https://imagepipelinestore.queue.core.windows.net/",
    "table": "https://imagepipelinestore.table.core.windows.net/"
  },
  "primaryLocation": "westus",
  "provisioningState": "Succeeded",
  "resourceGroup": "imagepipelinerg",
  "secondaryEndpoints": null,
  "secondaryLocation": null,
  "sku": {
    "name": "Standard_LRS",
    "tier": "Standard"
  },
  "statusOfPrimary": "Available",
  "statusOfSecondary": null,
  "tags": {},
  "type": "Microsoft.Storage/storageAccounts"
}
```

### Storage Account Credentials

A piece of information that will be needed by the applications we will deploy is the credentials for accessing the Storage Account.  To retrieve tthe Storage Account Keyso using the Azure CLI, the process is:

```bash
jims@dockeropolis:~$ az storage account keys list --name="imagepipelinestore" --resource-group="imagepipelinerg"
{
  "keys": [
    {
      "keyName": "key1",
      "permissions": "FULL",
      "value": "uPtltDA82dIm8JRWyvE5Tpu/uXB3aRFB5va7kssVj4huojgXFw6coCeTZ3ExgzwN9AHYzWCRKspgpP9RxfVCw=="
    },
    {
      "keyName": "key2",
      "permissions": "FULL",
      "value": "dFRh/oayuFkl6myNvfT4+LRTd0Ctg/Fp4d+s9C0y0t/sShl0eqgE1gqeqI1jYf92J6mSw0SyHuoQiys+ePdkEQ=="
    }
  ]
}
```

## Create the Azure Service Bus Queues

In order to create both of the Service Bus Queues, the Azure CLI will be used to deploy an Azure Resource Manager Template.  The link to the template is in the above defined values.  This step will require two JSON parameter files -- one for each Queue.

The content of each file are below:

```bash
jims@dockeropolis:~$ cat imagepipelineprocq.parameters.json 
{
    "serviceBusNamespaceName": {
        "value": "imagepipelinesbus"
    },
    "serviceBusQueueName": {
        "value": "imagepipelineprocq"
    }
}
```

```bash
jims@dockeropolis:~$ cat imagepipelinenotq.parameters.json 
{
    "serviceBusNamespaceName": {
        "value": "imagepipelinesbus"
    },
    "serviceBusQueueName": {
        "value": "imagepipelinenotq"
    }
}
```

The first template deployment will create both a Service Bus Namespace as well as the Service Bus Queue.  The second template deployment will create a second Service Bus Queue within the same Service Bus Namespace.  This is as desired.

To create each Queue using the Azure CLI template deployment, follow below.  One thing to **note**, the current Azure CLI has an inconsistency when specifying a file for `parameters`.  The CLI requires that the file name be prefaced with an `@` symbol.  However, `=` can not be used between `--parameters` and the file name.

For the Processing Queue:

```bash
jims@dockeropolis:~$ az resource group deployment create \
> --name="imagepipelineproc-dep" \
> --resource-group="imagepipelinerg" \
> --template-uri="https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-servicebus-queue/azuredeploy.json" \
> --parameters @./imagepipelineprocq.parameters.json
{
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.Resources/deployments/imagepipelineproc-dep",
  "name": "imagepipelineproc-dep",
  "properties": {
    "correlationId": "413ad9be-047d-4c45-904e-288b4366d8a8",
    "debugSetting": null,
    "dependencies": [
      {
        "dependsOn": [
          {
            "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.ServiceBus/namespaces/imagepipelinesbus",
            "resourceGroup": "imagepipelinerg",
            "resourceName": "imagepipelinesbus",
            "resourceType": "Microsoft.ServiceBus/namespaces"
          }
        ],
        "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.ServiceBus/namespaces/imagepipelinesbus/Queues/imagepipelineprocq",
        "resourceGroup": "imagepipelinerg",
        "resourceName": "imagepipelinesbus/imagepipelineprocq",
        "resourceType": "Microsoft.ServiceBus/namespaces/Queues"
      }
    ],
    "mode": "Incremental",
    "outputs": {},
    "parameters": {
      "serviceBusNamespaceName": {
        "type": "String",
        "value": "imagepipelinesbus"
      },
      "serviceBusQueueName": {
        "type": "String",
        "value": "imagepipelineprocq"
      }
    },
    "parametersLink": null,
    "providers": [
      {
        "id": null,
        "namespace": "Microsoft.ServiceBus",
        "registrationState": null,
        "resourceTypes": [
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              "westus"
            ],
            "properties": null,
            "resourceType": "namespaces"
          },
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              null
            ],
            "properties": null,
            "resourceType": "namespaces/Queues"
          }
        ]
      }
    ],
    "provisioningState": "Succeeded",
    "template": null,
    "templateLink": {
      "contentVersion": "1.0.0.0",
      "uri": "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-servicebus-queue/azuredeploy.json"
    },
    "timestamp": "2016-12-01T22:11:33.542956+00:00"
  },
  "resourceGroup": "imagepipelinerg"
}
```

For the Notification Queue:

```bash
jims@dockeropolis:~$ az resource group deployment create \
> --name="imagepipelinepnot-dep" \
> --resource-group="imagepipelinerg" \
> --template-uri="https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-servicebus-queue/azuredeploy.json" \
> --parameters @./imagepipelinenotq.parameters.json
{
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.Resources/deployments/imagepipelineproc-dep",
  "name": "imagepipelineproc-dep",
  "properties": {
    "correlationId": "9861c164-3140-43c8-b6ef-f5af9d669168",
    "debugSetting": null,
    "dependencies": [
      {
        "dependsOn": [
          {
            "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.ServiceBus/namespaces/imagepipelinesbus",
            "resourceGroup": "imagepipelinerg",
            "resourceName": "imagepipelinesbus",
            "resourceType": "Microsoft.ServiceBus/namespaces"
          }
        ],
        "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.ServiceBus/namespaces/imagepipelinesbus/Queues/imagepipelinenotq",
        "resourceGroup": "imagepipelinerg",
        "resourceName": "imagepipelinesbus/imagepipelinenotq",
        "resourceType": "Microsoft.ServiceBus/namespaces/Queues"
      }
    ],
    "mode": "Incremental",
    "outputs": {},
    "parameters": {
      "serviceBusNamespaceName": {
        "type": "String",
        "value": "imagepipelinesbus"
      },
      "serviceBusQueueName": {
        "type": "String",
        "value": "imagepipelinenotq"
      }
    },
    "parametersLink": null,
    "providers": [
      {
        "id": null,
        "namespace": "Microsoft.ServiceBus",
        "registrationState": null,
        "resourceTypes": [
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              "westus"
            ],
            "properties": null,
            "resourceType": "namespaces"
          },
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              null
            ],
            "properties": null,
            "resourceType": "namespaces/Queues"
          }
        ]
      }
    ],
    "provisioningState": "Succeeded",
    "template": null,
    "templateLink": {
      "contentVersion": "1.0.0.0",
      "uri": "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-servicebus-queue/azuredeploy.json"
    },
    "timestamp": "2016-12-01T22:17:39.021599+00:00"
  },
  "resourceGroup": "imagepipelinerg"
}
```

## Create the Azure Event Hub

Just like the Queues above, Azure CLI will be used to deploy an Azure Resource Manager Template for creating the Event Hub.  The link to the template is in the above defined values.  The JSON parameters file for the Event Hub is as follows:

```bash
jims@dockeropolis:~$ az resource group deployment create \
> --name="imagepipelineeh-dep" \
> --resource-group="imagepipelinerg" \
> --template-uri="https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/201-event-hubs-create-event-hub-and-consumer-group/azuredeploy.json" \
> --parameters @./imagepipelineehanalyze.parameters.json
{
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.Resources/deployments/imagepipelineeh-dep",
  "name": "imagepipelineeh-dep",
  "properties": {
    "correlationId": "4dab1520-d636-4723-a26c-9b293768391f",
    "debugSetting": null,
    "dependencies": [
      {
        "dependsOn": [
          {
            "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.EventHub/namespaces/imagepipelineehns",
            "resourceGroup": "imagepipelinerg",
            "resourceName": "imagepipelineehns",
            "resourceType": "Microsoft.EventHub/namespaces"
          }
        ],
        "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.EventHub/Namespaces/imagepipelineehns/EventHubs/imagepipelineeh",
        "resourceGroup": "imagepipelinerg",
        "resourceName": "imagepipelineehns/imagepipelineeh",
        "resourceType": "Microsoft.EventHub/Namespaces/EventHubs"
      },
      {
        "dependsOn": [
          {
            "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.EventHub/Namespaces/imagepipelineehns/EventHubs/imagepipelineeh",
            "resourceGroup": "imagepipelinerg",
            "resourceName": "imagepipelineehns/imagepipelineeh",
            "resourceType": "Microsoft.EventHub/Namespaces/EventHubs"
          }
        ],
        "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/imagepipelinerg/providers/Microsoft.EventHub/Namespaces/imagepipelineehns/EventHubs/imagepipelineeh/ConsumerGroups/imagepipelineanalyze",
        "resourceGroup": "imagepipelinerg",
        "resourceName": "imagepipelineehns/imagepipelineeh/imagepipelineanalyze",
        "resourceType": "Microsoft.EventHub/Namespaces/EventHubs/ConsumerGroups"
      }
    ],
    "mode": "Incremental",
    "outputs": {
      "namespaceConnectionString": {
        "type": "String",
        "value": "Endpoint=sb://imagepipelineehns.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=2f1hHmSPO3Y57rXjdAr+0tcQiZCp10TVFMuxhTSZX3k="
      },
      "sharedAccessPolicyPrimaryKey": {
        "type": "String",
        "value": "2f1hHmSPO3Y57rXjdAr+0tcQiZCp10TVFMuxhTSZX3k="
      }
    },
    "parameters": {
      "consumerGroupName": {
        "type": "String",
        "value": "imagepipelineanalyze"
      },
      "eventHubName": {
        "type": "String",
        "value": "imagepipelineeh"
      },
      "namespaceName": {
        "type": "String",
        "value": "imagepipelineehns"
      }
    },
    "parametersLink": null,
    "providers": [
      {
        "id": null,
        "namespace": "Microsoft.EventHub",
        "registrationState": null,
        "resourceTypes": [
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              "westus"
            ],
            "properties": null,
            "resourceType": "Namespaces"
          },
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              null
            ],
            "properties": null,
            "resourceType": "Namespaces/EventHubs"
          },
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              null
            ],
            "properties": null,
            "resourceType": "Namespaces/EventHubs/ConsumerGroups"
          }
        ]
      }
    ],
    "provisioningState": "Succeeded",
    "template": null,
    "templateLink": {
      "contentVersion": "1.0.0.0",
      "uri": "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/201-event-hubs-create-event-hub-and-consumer-group/azuredeploy.json"
    },
    "timestamp": "2016-12-01T22:36:14.773061+00:00"
  },
  "resourceGroup": "imagepipelinerg"
}
```

## Retrieve Azure Service Bus and Azure Event Hub Credentials

Unfortunately, there is not a way to retrieve the credentials for either Service Bus or Event Hub from the
Azure CLI.  In order to do this, one needs to go to the [Azure Portal](https://portal.azure.com).

Rather than showing all the steps in images, the process is as follows:

  1.  Log into the [portal](https://portal.azure.com)
  2.  Along the left column, click `Resource Groups`
  3.  In the field with the text `Filter by name`, enter the Resource Group name `imagepipelinerg`
  4.  In the list below, select `imagepipelinerg`
  5.  At this point, the portal should resemble:

![Resource Group Overview](https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/imagepipeline_on_deis/creds_resource_group_overview.png "Resource Group Overview")

  6.  Select the Service Bus `imagepipelinesbus`.  The portal will resemble:

![Service Bus Resource Overview](https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/imagepipeline_on_deis/creds_service_bus_overview.png "Service Bus Resource Overview")

  7.  Select `Shared Access Policy`, the portal should resemble:

![Shared Access Policy](https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/imagepipeline_on_deis/creds_service_bus_shared_access_policy.png "Shared Access Policy")

  8.  On this screen, select the Policy name.  It should be `RootManageSharedAccessKey`.
  9.  Click on `RootManageSharedAccessKey`.  This will open a pane showing the access keys as follows:

![Shared Access Policy Key](https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/imagepipeline_on_deis/creds_service_bus_shared_access_policy_key.png "Shared Access Policy Key")

  10.  Click the Clipboard icon next to the `Primary Key`, this will copy it to the clipboard.
  11.  Save the Policy/Name and Key somewhere.  

The above steps can be repeated for the Event Hub `imagepipelineehns`.

For the Azure Service Bus Namespace `imagepipelinesbus`, the values are:

  - Name / Policy:  `RootManageSharedAccessKey`
  - Key:  `d+efxLejTKA2vTAJB638XGPRDcc+PIyqWfBhO9D6IMg=`

For the Azure Event Hub `imagepipelineehns`, the values are:

  - Name / Policy:  `RootManageSharedAccessKey`
  - Key:  `2f1aFtPO3Y57rXjdAr+0btIiZCp10TVFMuxhREXX3k=`