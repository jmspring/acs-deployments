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
<Data Source> --> [Ingest] --> (Message into **Processing Queue**)
                     |
                     --> (Data into **Blob Store**)


(**Processing Queue**) --> [Data Analyzer] --> (Message into **Notification Queue**)
                            ^      |
         (**Blob Store**) --|      --> (Data Analysis Output into **Analysis Event Hub**)


(**Notification Queue**) --> [Notifier]
```

The components in the diagram translate as follows:

  - <Data Source> represents an external device from which data is received
  - [Ingest], [Data Analyzer], and [Notifier] are services running within Deis Workflow
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

For a tutorial of what Azure Templates are and how to work with them, please take a look [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-template-walkthrough).  

## Define the Values Needed for deployment

The application that will be deployed in the next walkthrough is a set of services for capturing and analyzing
images.  So, the names chosen below will reflect on that.

The first thing needing defining is the Resource Group.  The Resource Group will be created using the Azure CLI.  The values needed:

  - Resource Group Name:  imagepipelinerg
  - Location:  West US (for the CLI, the value is `westus`)

For the Azure Storage Account, the CLI will also be used to create the Storage Account.  There are four values needed:

  - Resource Group Name:  imagepipelinerg (**as above**)
  - Location:  West US (**as above**)
  - Storage Account Name:  imagepipelinestore
  - Storage Account SKU:  Standard_LRS (standard, locally redundant storage)

For the Azure Service Bus Queues, one of the existing [Azure Quickstart Templates](https://github.com/Azure/azure-quickstart-templates/tree/master/101-servicebus-queue).  If you examine
the [template deployment parameters](https://github.com/Azure/azure-quickstart-templates/blob/master/201-servicebus-create-queue/azuredeploy.parameters.json) the needed parameters (plus arguments for the CLI deployment command):

  - Resource Group Name:  imagepipelinerg (**as above**)
  - Deployment Names:
    - imagepipelineproc-dep (for the **Processing Queue**)
    - imagepipelinenot-dep (for the **Notification Queue**)
  - Service Bus Namespace:  imagepipelinesbus
  - Service Bus Queue Names:
    - imagepipelineprocq (for the **Processing Queue**)
    - imagepipelinenotq (for the **Notification Queue**)

For creating the Azure Event Hub, another [quickstart template](https://github.com/Azure/azure-quickstart-templates/tree/master/201-event-hubs-create-event-hub-and-consumer-group) is used.  The [parameters](https://github.com/Azure/azure-quickstart-templates/blob/master/201-event-hubs-create-event-hub-and-consumer-group/azuredeploy.parameters.json) required (plus arguments for the CLI deployment command) are:

  - Resource Group Name:  imagepipelinerg (**as above**)
  - Deployment Name:  imagepipelineeh-dep
  - Event Hub Namespace:  imagepipelineehns
  - Event Hub Name:  imagepipelineeh
  - Event Hub Consumer Group:  imagepipelineanalyze

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

