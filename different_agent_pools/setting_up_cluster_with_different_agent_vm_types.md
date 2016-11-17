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
            "keyData": ""
          }
        ]
      }
    }
  }
}

### Run acs-engine to generate ARM templates


