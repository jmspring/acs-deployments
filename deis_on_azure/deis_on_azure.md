# Installing DEIS on Azure

With the inclusion of Kubernetes as part of the (Azure Container Service)[https://azure.microsoft.com/en-us/services/container-service/], 
the deployment of DEIS onto Azure has become significantly easier.  The basic steps to deploy DEIS into Azure
are as follows:

    - Install the (Azure CLI)[https://github.com/Azure/azure-cli]
    - Log in to your Azure Account
    - Create a Service Principal
    - Create the Kubernetes Cluster using the CLI tools
    - Install and Configure `kubectl`
    - Install (DEIS Workflow)[https://deis.com/docs/workflow/]
    - Verify everything is running  

## Installing the Azure CLI

The (Azure CLI)[https://github.com/Azure/azure-cli] (also known as Azure CLI 2.0 Preview) is the recommended way
to interact with the Azure Container Service.  It has support for the Kubernetes Orchestrator.

Installation is documented on the github repository.  Options include native install (Windows, Linux, and OSX)
via **pip** or (for OSX / Linux) an install script, or even a Docker container.  Pick the method most appropriate
to your environment.

## Log in to your Azure Account

Once you have the tools installed, you need an (Azure account)[https://azure.microsoft.com/en-us/free/].  You
will also need to know your **username** and **password** to login.  To login with the CLI:

```bash
jims@dockeropolis:~$ az login
To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code BPM3W42L8 to authenticate.
[
  {
    "cloudName": "AzureCloud",
    "id": "abf7ec88-8e28-41ed-8537-5e17766001f5",
    "isDefault": true,
    "name": "Developers Research",
    "state": "Enabled",
    "tenantId": "abf988bf-86f1-41af-91ab-2d7cd011db47",
    "user": {
      "name": "jaspring@microsoft.com",
      "type": "user"
    }
  }
]
```

The device login screen looks like [this](https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/device_login.png).

## Create a Service Principal

A Service Principal is an entity that is granted rights to perform various actions within a subscription.
For Kubernetes on Azure, a Service Principal is needed because the Kubernetes Azure Provider uses it to 
manipulate or create or destroy resources as needed.

In the Azure CLI 2.0 Preview, there are commands to simplify the creation of a Service Principal.  As of
version (v0.1.0b9)[https://github.com/Azure/azure-cli/releases/tag/all-v0.1.0b9], the single command for
creatinig a Service Principal is a bit flakey.  Below are the steps needed, they do involve knowing a bit
more about your account information.  Below will walk you through the needed information.  

To avoid having to understand too much about Azure Active Directory, basic explanations of each piece 
are given.  It's easiest to think of a Service Principal as two pieces:

    - An "application" which is an entry in Active Directory that has both a name and a password.  
      It will also have an Object Identifier (GUID) associated with it.
    - A "service principal" which is an entry that ties your "application" and is given permissions 
      to being able to perform certain actions within your subscription.


To create the Service Principal, you will need to determine:

    - A name for the application.  This will be used in multiple parts of the application 
      creation command line.
    - A password for the application.

The steps are:

    - Create the Application
    - Create the Service Principal, associating it with the Application
    - Grant permissions to the Service Principal

For this Service Principal, the following values are used:

    - name: deisonk8sapp
    - password: DeisAndK8sPlayWell

```bash
jims@dockeropolis:~$ az ad app create \
> --display-name="deisonk8sapp" \
> --homepage="http://localhost/deisonk8sapp" \ 
> --identifier-uris="http://localhost/deisonk8sapp" 
> --key-type="Password" 
> --password="DeisAndK8sPlayWell"
{
  "appId": "29f7912c-1f26-4b85-9d2d-7f627415276b",
  "appPermissions": null,
  "availableToOtherTenants": false,
  "displayName": "deisonk8sapp",
  "homepage": "http://localhost/deisonk8sapp",
  "identifierUris": [
    "http://localhost/deisonk8sapp"
  ],
  "objectId": "21c4a766-c220-48b9-9df9-956bf45c71de",
  "objectType": "Application",
  "replyUrls": []
}
```

From the output of the `az ad app create` command, the next command will use the `appId`
to associate the Application with the Service Principal.

```bash
jims@dockeropolis:~$ az ad sp create \
> --id="29f7912c-1f26-4b85-9d2d-7f627415276b"
{
  "appId": "29f7912c-1f26-4b85-9d2d-7f627415276b",
  "displayName": "deisonk8sapp",
  "objectId": "2d054033-4ca4-4f4f-85e0-eaa701ee8caf",
  "objectType": "ServicePrincipal",
  "servicePrincipalNames": [
    "29f7912c-1f26-4b85-9d2d-7f627415276b",
    "http://localhost/deisonk8sapp"
  ]
}
```

At this point, the Service Principal needs to be granted rights.  For this, you will need 
to create a role.  Two pieces of information for creating the role are needed.  First is
the `appId` from the `az ad sp create` command above.  The second is the `id` also 
referred to as the `subscriptionId` which was present in the output of the `az login` 
command executed in the **Log in to your Azure Account** section.

```bash
jims@dockeropolis:~$ az role assignment create --role="Contributor" --assignee="29f7912c-1f26-4b85-9d2d-7f627415276b" --scope="/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5"
{
  "id": "/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5/providers/Microsoft.Authorization/roleAssignments/0029a568-e95f-41b0-9a83-73202beb03ab",
  "name": "0029a568-e95f-41b0-9a83-73202beb03ab",
  "properties": {
    "principalId": "2d054033-4ca4-4f4f-85e0-eaa701ee8caf",
    "roleDefinitionId": "/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c",
    "scope": "/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5"
  },
  "type": "Microsoft.Authorization/roleAssignments"
}
```

Now the creation of the Service Principal is complete.  The two pieces of information needed, 
for the Service Principal, later in this tutorial from above are:

    - appId: 29f7912c-1f26-4b85-9d2d-7f627415276b
    - password: DeisAndK8sPlayWell

## Create the Kubernetes Cluster using the CLI tools

For creating the Kubernetes Cluster on Azure, there are two approaches.  The first, and simplest, is
to use the Azure Container Service Resource Provider (through the portal or through the CLI.  The 
second is through the open-sources (acs-engine)[https://github.com/Azure/acs-engine] which 
allows you to customize the templates generated.

The standard deployment that is generated (as well as additional instructions for using acs-engine for
Kubernetes) can be found (here)[https://github.com/Azure/acs-engine/blob/master/docs/kubernetes.md].

For creating the cluster, we will first need to create a resource group.  For this example, we will need
both a name for the reesource group and a region within the Azure Cloud to create it:

    - resource-group: "deisonk8srg"
    - location: "westus"

Creating the resource group:

```bash
jims@dockeropolis:~$ az resource group create \
> --name="deisonk8srg" \
> --location="westus"
{
  "id": "/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg",
  "location": "westus",
  "managedBy": null,
  "name": "deisonk8srg",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null
}
```

To create the cluster, additional parameters are needed.

Values needed from above:

    - resource-group: deisonk8srg
    - location: westus
    - service-principal: 29f7912c-1f26-4b85-9d2d-7f627415276b
    - client-secret: DeisAndK8sPlayWell

Additionally, we need information about the cluster itself.  DEIS requires the **kubernetes** 
**orchestrator**.  For now, it is recommended to use a single master, as HA in Kubernetes 
masters is not yet supported.  The list of values used in this walk through are as follows:

    - orchestrator: kubernetes
    - master-count: 1
    - agent-count: 4
    - agent-vm-size: Standard_D2_v2 (2 core, 7gb of ram)

Additionally, for naming and administrative purposes, we need to name the admin, the cluster,
a dns prefix (for public facing resources) and a path to the SSH public key file (of the key
pair that will be used to access the cluster):

    - admin-username: dadmin
    - name: k8sanddeis
    - dns-prefix: k8sanddeis
    - ssh-key-value file: /home/jims/id_acs_rsa.pub

This results in the following command running:

```bash
jims@dockeropolis:~$ az acs create \
> --resource-group="deisonk8srg" \
> --location="westus" \
> --service-principal="29f7912c-1f26-4b85-9d2d-7f627415276b" \
> --client-secret="DeisPlaysWithK8s" \
> --orchestrator-type=kubernetes \
> --master-count=1 \
> --agent-count=4 \
> --agent-vm-size="Standard_D2_v2" \
> --admin-username="dadmin" \
> --name="k8sanddeis" \
> --dns-prefix="k8sanddeis" \
> --ssh-key-value @/home/jims/.ssh/id_acs_rsa.pub
waiting for AAD role to propogate.done
{
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Resources/deployments/azurecli1479772992.8962212",
  "name": "azurecli1479772992.8962212",
  "properties": {
    "correlationId": "2f7ca0b6-b475-43d1-9a19-bbe558b9cee8",
    "debugSetting": null,
    "dependencies": [],
    "mode": "Incremental",
    "outputs": null,
    "parameters": null,
    "parametersLink": null,
    "providers": [
      {
        "id": null,
        "namespace": "Microsoft.ContainerService",
        "registrationState": null,
        "resourceTypes": [
          {
            "aliases": null,
            "apiVersions": null,
            "locations": [
              "westus"
            ],
            "properties": null,
            "resourceType": "containerServices"
          }
        ]
      }
    ],
    "provisioningState": "Succeeded",
    "template": null,
    "templateLink": null,
    "timestamp": "2016-11-22T00:08:21.619001+00:00"
  },
  "resourceGroup": "deisonk8srg"
}
```

At this point, the Kubernetes cluster is up and running.  In order to interact with it from
your local machine, you will need to install and configure `kubectl`.