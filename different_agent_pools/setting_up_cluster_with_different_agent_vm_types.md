# A DC/OS Cluster with Multiple Private Agent Pools

This write up explains how to use the [acs-engine](https://github.com/Azure/acs-engine) to
deploy a DC/OS cluster which contains multiple private agent pools.  This will allow for 
each agent pool to have different VM types for different work loads.

## Prerequisites:

In order to complete this deployment, you need the following:

    - An Azure subscription (as well as knowing the subscription ID)
    - *acs-engine* either built or the docker container, information is [here](https://github.com/Azure/acs-engine/blob/master/docs/acsengine.md)
    - The Azure CLI, either (azure-xplat-cli)[https://github.com/Azure/azure-xplat-cli] or (azure cli)[https://github.com/Azure/azure-cli]

For specifics about the cluster, you will need to determine the following:

    - A resource group and location for the entirety of the deployment.
    - For the VNet:
        - A name for the VNet (vnetName)
        - Determine number of subnets
        - Determine addressPrefixes (ranges) for each subnet (vnetSubnet1Prefix ... )
        - A name for each subnet (vnetSubnet1Name ...)
    - For each pool of nodes (master, public, private 1, and private 2) you will need:
        - A name for the pool (poolName)
        - The number of nodes (poolCount)
        - The type of VM for each pooll (poolVmType)
        - Which subnet it will belong to based on VNet definition (poolSubnet)
    - For the master pool and any public pool, you will need
        - A name prefix (poolDnsPrefix)
    - For the master pool an initial address within the master subnet is needed
      as a base IP address to start with when assigning IPs to the master nodes.
      (poolFirstStaticIP)

For purposes of showing these values in the template, the variable names above will be 
prefixed with "master", "publicAgent", "privateAgentN" in the template. 

The overall deployment will look like ![this](https://raw.githubusercontent.com/jmspring/acs-deployments/master/different_agent_pools/acs-dcos-vnet.png)

## The Process

Using the above information, the process for deploying the desired cluster is as follows:

    - Setup the VNet
    - Setup acs-engine template file
    - Run acs-engine to generate ARM templates
    - Deploy the ARM templates into Azure

### Setup the VNet

The base ARM Template for setting up the VNet is below.  Values demarcated by "<" and ">" 
refer to the variable names mentioned above.  The elipses ("...") are there in the case 
more than two subnets are specified.

```javascript
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {  },
  "variables": {  },
  "resources": [
    {
      "apiVersion": "2016-03-30",
      "location": "[resourceGroup().location]",
      "name": "<vnetName>",
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "<vnetSubnet1Prefix>",
            "<vnetSubnet2Prefix>".
            ...
          ]
        },
        "subnets": [
          {
            "name": "vnetSubnet1Name>",
            "properties": {
              "addressPrefix": "<vnetSubnet1Prefix>"
            }
          },
          {
            "name": "vnetSubnet2Name>",
            "properties": {
              "addressPrefix": "<vnetSubnet2Prefix>"
            }
          },
          ...
        ]
      },
      "type": "Microsoft.Network/virtualNetworks"
    }
  ]
}
```

### Setup acs-engine template file

In the acs-engine template file, one takes the values determined above and fills out the 
appropriate information within the template.  In addition to the information on the VNet 
and the pools, the account **Subscription ID** will be needed as well as the **Resource Group**
the VNet was deployed into.

As mentioned previously, this deployment assumes three subnets:

    - masterPoolSubnet
    - publicAgentPoolSubnet
    - privateAgentPoolSubnet

As such, the acs-engine template file would resemble:

```javascript
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "DCOS"
    },
    "masterProfile": {
      "count": <masterPoolCount>,
      "dnsPrefix": "<masterPoolDnsPrefix>",
      "vmSize": "<masterPoolVmType>",
      "vnetSubnetId": "/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<masterPoolSubnetName>",
      "firstConsecutiveStaticIP": "<masterPoolFirstStaticIP>" 
    },
    "agentPoolProfiles": [
      {
        "name": "<privateAgentPool1Name>",
        "count": "<privateAgentPool1Count>",
        "vmSize": "<privateAgentPool1VmType>",
        "availabilityProfile": "AvailabilitySet",
        "vnetSubnetId": "/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<privateAgentPoolSubnetName>",
      },
      {
        "name": "<privateAgentPool2Name>",
        "count": "<privateAgentPool2Count>",
        "vmSize": "<privateAgentPool2VmType>",
        "availabilityProfile": "AvailabilitySet",
        "vnetSubnetId": "/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<privateAgentPoolSubnetName>",
      },
      {
        "name": "<publicAgentPoolName>",
        "count": "<publicAgentPoolCount>",
        "vmSize": "<publicAgentPoolVmType>",
        "dnsPrefix": "<publicAgentPoolDnsPrefix>",
        "vnetSubnetId": "/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<publicAgentPoolSubnetName>",
        "ports": [
          80,
          443,
          8080
        ]
      }
    ],
    "linuxProfile": {
      "adminUsername": "azureuser",
      "ssh": {
        "publicKeys": [
          {
            "keyData": "<sshKeyData>"
          }
        ]
      }
    }
  }
}
```

### Run acs-engine to generate ARM templates

When running acs-engine, we will assume the following directory structure and files:

```
- cluster_setup/
  |-- config/
      |-- acs-parameters.json
      |-- vnet-template.json
  |-- output/
      |-- acs-engine/
```

To run:

```bash
jims@foo:~/cluster_setup$ acs-engine -artifcates ./output/acs-engine ./config/acs-parameters.json
```

This will read information from the acs-parameters.json file and generate the necessary configuration 
and templates to deploy the DC/OS cluster.

Upon running, the following files are in the output directory:

```bash
jims@foo:~/cluster_setup$ ls -l output/
total 108
-rw------- 1 jims jims   2470 Nov 17 10:44 apimodel.json
-rw------- 1 jims jims 100596 Nov 17 10:44 azuredeploy.json
-rw------- 1 jims jims   1905 Nov 17 10:44 azuredeploy.parameters.json
```

### Deploy the ARM templates into Azure

To deploy the ARM templates, as mentioned previously, one will need an Azure account and the 
Azure CLI tools installed.  In this case, the (azure-xplat-cli)[https://github.com/Azure/azure-xplat-cli]
will be used.

In addition, one will need:

    - The name of a resource group
    - The location to create and deploy resources into the resource group

To deploy the templates, follow the following steps:

    - Log in to Azure
    - Create the resource group
    - Deploy the VNet template
    - Deploy the DC/OS Cluster template created by acs-engine

To log in to Azure:

```bash
jims@foo:~/cluster_setup$ azure login
info:    Executing command login
|info:    To sign in, use a web browser to open the page https://aka.ms/devicelogin. Enter the code B6LMXDFRE to authenticate.
|info:    Added subscription Research Playground
info:    Setting subscription "Research Playground" as default
+
info:    login command OK
```

To create the resource group, we will use the resource group name "jmsdcosrg" and deploy
into West US 2.

```bash
jims@foo:~/cluster_setup$ azure group create jmsdcosrg westus2
info:    Executing command group create
+ Getting resource group jmsdcosrg                                             
+ Creating resource group jmsdcosrg                                            
info:    Created resource group jmsdcosrg
data:    Id:                  /subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/jmsdcosrg
data:    Name:                jmsdcosrg
data:    Location:            westus2
data:    Provisioning State:  Succeeded
data:    Tags: null
data:    
info:    group create command OK
```

Deployments can have names, in the case of deploying the vnet template, the deployment will be named
"vnet-dep".  To deploy the VNet template into the resource group:

```bash
jims@foo:~/cluster_setup$ azure group deployment create --name "vnet-dep" --resource-group jmsdcosrg --template-file ./config/vnet-template.json 
info:    Executing command group deployment create
info:    Supply values for the following parameters
+ Initializing template configurations and parameters                          
+ Creating a deployment                                                        
info:    Created template deployment "vnet-dep"
+ Waiting for deployment to complete                                           
+                                                                              
info:    Resource 'jmsdcosvnet' of type 'Microsoft.Network/virtualNetworks' provisioning status is Running
+                                                                              
...
+                                                                              
info:    Resource 'jmsdcosvnet' of type 'Microsoft.Network/virtualNetworks' provisioning status is Succeeded
data:    DeploymentName     : vnet-dep
data:    ResourceGroupName  : jmsdcosrg
data:    ProvisioningState  : Succeeded
data:    Timestamp          : 2016-11-17T19:29:01.320Z
data:    Mode               : Incremental
data:    CorrelationId      : 29f4316d-d4f6-4a57-bc2b-ac238afb5460
info:    group deployment create command OK
```

And finally, deploying the DC/OS cluster into the resource group and using the VNet just created 
(note - the deployment is quite verbose and all output will not be shown):

```
