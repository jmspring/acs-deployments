# Installing Deis on Azure

With the inclusion of Kubernetes as part of the [Azure Container Service](https://azure.microsoft.com/en-us/services/container-service/), 
the deployment of Deis, using the [Deis Workflow](https://github.com/deis/workflow) onto Azure has become 
significantly easier.  The basic steps to deploy Deis into Azure are as follows:

  - Install the Azure CLI
  - Log in to your Azure Account
  - Create a Service Principal (optional)
  - Create the Kubernetes Cluster
    - Using the CLI tools
    - Using acs-engine 
  - Install and Configure `kubectl`
  - Install Deis Workflow
  - Verify everything is running  

## Installing the Azure CLI

The [Azure CLI](https://github.com/Azure/azure-cli) (also known as Azure CLI 2.0 Preview) is the recommended way
to interact with the Azure Container Service.  It has support for the Kubernetes Orchestrator.

Installation is documented on the github repository.  Options include native install (Windows, Linux, and OSX)
via **pip** or (for OSX / Linux) an install script, or even a Docker container.  Pick the method most appropriate
to your environment.

## Log in to your Azure Account

Once you have the tools installed, you need an [Azure account](https://azure.microsoft.com/en-us/free/).  You
will also need to know your **username** and **password** to login.  To login with the CLI:

```bash
jims@dockeropolis:~$ az login
To sign in, use a web browser to open the page [https://aka.ms/devicelogin] and enter the code BPM3W42L8 to authenticate.
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

When using the Azure Portal, a Service Principal is required.  What a Service Principal is and how to 
create one is outlined below.  In this tutorial, since it uses the Azure CLI, specifying a Service 
Principal is optional.  If one is not specified, it will be created as part of the deployment.

A Service Principal is an entity that is granted rights to perform various actions within a subscription.
For Kubernetes on Azure, a Service Principal is needed because the Kubernetes Azure Provider uses it to 
manipulate or create or destroy resources as needed.

In the Azure CLI 2.0 Preview, there are commands to simplify the creation of a Service Principal.  As of
version [v0.1.0b9](https://github.com/Azure/azure-cli/releases/tag/all-v0.1.0b9), the single command for
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
> --identifier-uris="http://localhost/deisonk8sapp" \ 
> --key-type="Password" \
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

## Create the Kubernetes Cluster

For creating the Kubernetes Cluster on Azure, there are two approaches.  The first, and simplest, is
to use the Azure Container Service Resource Provider (through the portal or through the CLI.  The 
second is through the open-sources [acs-engine](https://github.com/Azure/acs-engine) which 
allows you to customize the templates generated.

The standard deployment that is generated (as well as additional instructions for using acs-engine for
Kubernetes) can be found [here](https://github.com/Azure/acs-engine/blob/master/docs/kubernetes.md).  A 
brief walk through using `acs-engine` is included below.

For each deployment method, you will need to create a resource group.  For this example, we will need
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

For each deployment method below, additional parameters are needed.  

Values needed from the resource group:

  - resource-group: deisonk8srg
  - location: westus

The `acs-engine` deployment requires a Service Principal.  The CLI deployment using the 
ACS Resource Provider can either specity a Service Principal or one will be created.  In
the case where a Service Principal is specified, then you will also need the following values.  

  - service-principal: 29f7912c-1f26-4b85-9d2d-7f627415276b
  - client-secret: DeisAndK8sPlayWell

Additionally, we need information about the cluster itself.  Deis requires the **kubernetes** 
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

For the `acs-engine` deployment approach, the contents of `/home/jims/id_acs_rsa.pub` will 
be used.

Once the resource group is created, decide which deployment method is desirable.  Use the
values above to deploy the cluster.

### Create the Kubernetes Cluster using the CLI tools

**NOTE** In a couple of deployments attempting to create Deis applications resulted in an Internal 
Server Error due to validation issues of the Kubernetes apiserver certificate.  For now, it is 
recommended to use the **Create the Kubernetes Cluster using acs-engine** approach below.

Using the parameters specified above, you run one of the following two CLI commands depending on 
if you are specifying the Service Principal or you are not.

Without a Service Principal:

```bash
jims@dockeropolis:~$ az acs create \
> --resource-group="deisonk8srg" \
> --location="westus" \
> --orchestrator-type=kubernetes \
> --master-count=1 \
> --agent-count=4 \
> --agent-vm-size="Standard_D2_v2" \
> --admin-username="dadmin" \
> --name="k8sanddeis" \
> --dns-prefix="k8sanddeis" \
> --ssh-key-value @/home/jims/.ssh/id_acs_rsa.pub
```

Or with a Service Principal:

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
```

The result for both will resemble:

```bash
waiting for AAD role to propogate.done
{
  "id": "/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Resources/deployments/azurecli1479772992.8962212",
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

### Create the Kubernetes Cluster using acs-engine

Another approach to deploying a Kubernetes cluster on to Azure is to use the 
[acs-engine](https://github.com/Azure/acs-engine/blob/master/docs/kubernetes.md).  Rather 
than going completely through the linked tutorial, this section assumes:

  - `acs-engine` has been pulled and installed locally
  - The cluster deployed through `acs-engine` will use the same properties as the one deployed in the prior section.

When using `acs-engine` for a vanilla deployment, the process is pretty simple:

  - Fill out an `acs-engine` parameters file.
  - Run `acs-engine` on the parameters file.
  - Use the Azure CLI to deploy the generated template.

The base Kubernetes parameters file is:

```bash
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "Kubernetes"
    },
    "masterProfile": {
      "count": 1,
      "dnsPrefix": "",
      "vmSize": "Standard_D2_v2"
    },
    "agentPoolProfiles": [
      {
        "name": "agentpool1",
        "count": 3,
        "vmSize": "Standard_D2_v2",
        "availabilityProfile": "AvailabilitySet"
      }
    ],
    "linuxProfile": {
      "adminUsername": "",
      "ssh": {
        "publicKeys": [
          {
            "keyData": ""
          }
        ]
      }
    },
    "servicePrincipalProfile": {
      "servicePrincipalClientID": "",
      "servicePrincipalClientSecret": ""
    }
  }
}
```

Using the parameters previously specified, a filled out parameters file will look like:

```bash
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "Kubernetes"
    },
    "masterProfile": {
      "count": 1,
      "dnsPrefix": "k8sanddeis",
      "vmSize": "Standard_D2_v2"
    },
    "agentPoolProfiles": [
      {
        "name": "agentpool1",
        "count": 4,
        "vmSize": "Standard_D2_v2",
        "availabilityProfile": "AvailabilitySet"
      }
    ],
    "linuxProfile": {
      "adminUsername": "dadmin",
      "ssh": {
        "publicKeys": [
          {
            "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5GaGb48t3eNRt6HNkiRGhneNPs1m0o2TtXmVUFyqPn5x5k/PUnJib0PSS/0LL8HrkKuDdVlEIkojROya9OD5cwoe/xfjxjFXhNqsP8F/NY57b6M1KGpX1cq8zv3M2HTQWwnmGwImG3/UM7qbi/4Z5EZ3lCngEfYsio6
cmtpb3EI/A8WAAHj5SD0EgYajznmkCiEn2ls7Nf94t8PdXxZUpCuHagpG+8T61ysbO/qDDTvw2c3LHttfOtPLQrKwA/k6n9kHDTyDZ22KUJDD6s0TvzqQhsYVGo640Y7+qkgUy+vsvzFDOhShjrMEY+g6XbQDpTt3KaDfHGKpYwBo6UumT jims@dockeropolis"
          }
        ]
      }
    },
    "servicePrincipalProfile": {
      "servicePrincipalClientID": "29f7912c-1f26-4b85-9d2d-7f627415276b",
      "servicePrincipalClientSecret": "DeisAndK8sPlayWell"
    }
  }
}
```

Assume this file is `/home/jims/kubernetes.json`.

Given the above parameters file, use `acs-engine` to generate the required templates.  The target
output directory for the templates will be `/home/jims/acs-out/`.  Given that, the command line for
generating the configuration files is:

```bash
jims@dockeropolis:~$/ acs-engine -artifacts /home/jims/acs-out` /home/jims/kubernetes.json
cert creation took 9.142963463s
wrote /home/jims/acs-out/apimodel.json
wrote /home/jims/acs-out/azuredeploy.json
wrote /home/jims/acs-out/azuredeploy.parameters.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.australiaeast.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.australiasoutheast.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.brazilsouth.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.canadacentral.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.canadaeast.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.centralindia.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.centralus.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.eastasia.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.eastus.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.eastus2.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.japaneast.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.japanwest.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.northcentralus.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.northeurope.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.southcentralus.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.southeastasia.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.southindia.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.uksouth.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.ukwest.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.westcentralus.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.westeurope.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.westindia.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.westus.json
wrote /home/jims/acs-out/kubeconfig/kubeconfig.westus2.json
wrote /home/jims/acs-out/ca.key
wrote /home/jims/acs-out/ca.crt
wrote /home/jims/acs-out/apiserver.key
wrote /home/jims/acs-out/apiserver.crt
wrote /home/jims/acs-out/client.key
wrote /home/jims/acs-out/client.crt
wrote /home/jims/acs-out/kubectlClient.key
wrote /home/jims/acs-out/kubectlClient.crt
acsengine took 9.172658455s
```

`acs-engine` creates the needed templates and other ancilary files.  To deploy the template 
created, the relevant files are `/home/jims/acs-out/azuredeploy.json` and 
`/home/jims/acs-out/azuredeploy.parameters.json`.  The command to deploy the templates is as
follows:

```bash
az resource group deployment create \
> --name "deisonk8s-dep" \
> --resource-group="deisonk8srg" \
> --template-file /home/jims/acs-out/azuredeploy.json \
> --parameters @/home/jims/acs-out/azuredeploy.parameters.json
{
  "id": "/subscriptions/abf7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Resources/deployments/deisonk8s-dep",
  "name": "deisonk8s-dep",
  "properties": {
    "correlationId": "f212d96c-de23-4e3c-aba0-4e1d14f95353",
    "debugSetting": null,
    "dependencies": [
    ...
    ],
    "provisioningState": "Succeeded",
    "template": null,
    "templateLink": null,
    "timestamp": "2016-11-24T00:26:28.144455+00:00"
  },
  "resourceGroup": "deisonk8srg"
}
```

Note the full output from above is quite long `...` implies missing content.  The actual output can be seen [here](https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/acs-engine_deploy_output.json).

## Install and Configure kubectl

At this point, the Kubernetes cluster is up and running.  In order to interact with it from
your local machine, you will need to install and configure `kubectl`.

[kubectl](http://kubernetes.io/docs/getting-started-guides/kubectl/) is a command-line tool for interacting
with your Kubernetes cluster.  For this example, `kubectl` will be installed in the bin directory within
the existing home directory.  Prebuilt binaries mentioned in the link above will be used.

```bash
jims@dockeropolis:~$ wget https://storage.googleapis.com/kubernetes-release/release/v1.4.4/bin/linux/amd64/kubectl
--2016-11-21 17:55:55--  https://storage.googleapis.com/kubernetes-release/release/v1.4.4/bin/linux/amd64/kubectl
Resolving storage.googleapis.com (storage.googleapis.com)... 216.58.194.176, 2607:f8b0:4005:804::2010
Connecting to storage.googleapis.com (storage.googleapis.com)|216.58.194.176|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 83220624 (79M) [application/octet-stream]
Saving to: ‘kubectl’

kubectl                      100%[=============================================>]  79.37M  1.31MB/s    in 2m 18s  

2016-11-21 17:58:14 (590 KB/s) - ‘kubectl’ saved [83220624/83220624]

jims@dockeropolis:~$ chmod +x kubectl
jims@dockeropolis:~$ mv kubectl ./bin/
jims@dockeropolis:~$ kubectl version
Client Version: version.Info{Major:"1", Minor:"4", GitVersion:"v1.4.4", GitCommit:"3b417cc4ccd1b8f38ff9ec96bb50a81ca0ea9d56", GitTreeState:"clean", BuildDate:"2016-10-21T02:48:38Z", GoVersion:"go1.6.3", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

At this point, `kubectl` is installed.  Each Kubernetes cluster has a config file which `kubectl` uses.
When a Kubernetes cluster is deployed via the Azure Container Service Resource Provider, the config can
be found within on the master node within the home directory of the admin user at `~/.kube/config`.

To retrieve the configuration and bring it locally, we can do the following:

```bash
jims@dockeropolis:~$ MASTER_K8S_IP=`az network public-ip list --resource-group deisonk8srg | jq -r '.[0].ipAddress'`
jims@dockeropolis:~$ scp -i ~/.ssh/id_acs_rsa dadmin@$MASTER_K8S_IP:.kube/config ./$MASTER_K8S_IP.kubeconfig
config                                                                           100% 6293     6.2KB/s   00:00
jims@dockeropolis:~$ export KUBECONFIG=`pwd`/$MASTER_K8S_IP.kubeconfig
```

To verify the cluster is up and healthy:

```bash
jims@dockeropolis:~$ kubectl cluster-info
Kubernetes master is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com
Heapster is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/heapster
KubeDNS is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
jims@dockeropolis:~$ kubectl get nodes
NAME                    STATUS                     AGE
k8s-agent-d84d22f6-0    Ready                      1h
k8s-agent-d84d22f6-1    Ready                      1h
k8s-agent-d84d22f6-2    Ready                      1h
k8s-agent-d84d22f6-3    Ready                      1h
k8s-master-d84d22f6-0   Ready,SchedulingDisabled   1h
```

At this point, our cluster should be up and running and healthy.  And we have `kubectl` configured
for accessing the cluster on our local machine.

## Install Deis Workflow

This tutorial is meant to go through all the steps to get Deis installed on Kubernetes on Azure.
It is not meant to go through every intricacy or the design concepts or components.  With that 
said, below, the tutorial follows the [Deis Workflow Quickstart Guide](https://deis.com/docs/workflow/quickstart/).

The few steps are:

  - Install CLI tools for Helm Classic and Deis Workflow
  - Install Deis Workflow on the Kubernetes Cluster

### Install CLI tools for Helm Classic and Deis Workflow

The instructions for the installation steps below are found [here](https://deis.com/docs/workflow/quickstart/install-cli-tools/).

For both Deis and Helm, the binaries will be installed into the same location as `kubectl`, which is the
`~/bin` directory.

Install Deis:

```bash
jims@dockeropolis:~$ curl -sSL http://deis.io/deis-cli/install-v2.sh | bash
Downloading deis-stable-linux-amd64 From Google Cloud Storage...

deis is now available in your current directory.

To learn more about deis, execute:

    $ ./deis --help

jims@dockeropolis:~$ mv deis ~/bin/
jims@dockeropolis:~$ deis version
v2.9.1
```

Install Helm:

```bash
jims@dockeropolis:~$ wget https://kubernetes-helm.storage.googleapis.com/helm-v2.0.0-linux-amd64.tar.gz
--2016-12-07 10:11:33--  https://kubernetes-helm.storage.googleapis.com/helm-v2.0.0-linux-amd64.tar.gz
Resolving kubernetes-helm.storage.googleapis.com (kubernetes-helm.storage.googleapis.com)... 216.58.216.176, 2607:f8b0:400a:807::2010
Connecting to kubernetes-helm.storage.googleapis.com (kubernetes-helm.storage.googleapis.com)|216.58.216.176|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14111189 (13M) [application/x-tar]
Saving to: ‘helm-v2.0.0-linux-amd64.tar.gz’

helm-v2.0.0-linux-amd64.tar. 100%[=============================================>]  13.46M  11.3MB/s    in 1.2s    

2016-12-07 10:11:35 (11.3 MB/s) - ‘helm-v2.0.0-linux-amd64.tar.gz’ saved [14111189/14111189]

jims@dockeropolis:~$ tar -zxvf helm-v2.0.0-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
jims@dockeropolis:$ cp linux-amd64/helm ~/bin
jims@dockeropolis:~$ helm version
Client: &version.Version{SemVer:"v2.0.0", GitCommit:"51bdad42756dfaf3234f53ef3d3cb6bcd94144c2", GitTreeState:"clean"}
Error: cannot connect to Tiller
```

Note, that Helm requires that you run an [initialization] (https://github.com/kubernetes/helm/blob/master/docs/quickstart.md) 
once per Kubernetes cluster.  To see which cluster Helm will install Tiller into, check your
Kubernetes config:

```bash
jims@dockeropolis:~$ kubectl config current-context
k8sanddeis
```

Initializing Helm:

```bash
jims@dockeropolis:~/go/src/github.com/Azure/acs-engine/examples/hackfest$ helm init
Creating /home/jims/.helm 
Creating /home/jims/.helm/repository 
Creating /home/jims/.helm/repository/cache 
Creating /home/jims/.helm/repository/local 
Creating /home/jims/.helm/repository/repositories.yaml 
Creating /home/jims/.helm/repository/local/index.yaml 
$HELM_HOME has been configured at $HOME/.helm.

Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!
```

At this point one is ready to install workflow.

### Install Deis Workflow on the Kubernetes Cluster

Since there is already an existing Kubernetes cluster, the path for installing Deis Workflow
follows the [Vagrant procress](https://deis.com/docs/workflow/quickstart/provider/vagrant/install-vagrant/).

First step is to make sure `helm` finds the config file for `kubectl`.  **Note** - if the terminal 
session was logged out of or restarted or did anything to disrupt the value of `KUBECONFIG` configured
above, the appropriate steps will be needed to ensure it is configured correctly.

```bash
jims@dockeropolis:~$ helm version
Client: &version.Version{SemVer:"v2.0.0", GitCommit:"51bdad42756dfaf3234f53ef3d3cb6bcd94144c2", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.0.0", GitCommit:"51bdad42756dfaf3234f53ef3d3cb6bcd94144c2", GitTreeState:"clean"}
```

Next step is to add the Deis chart repository:

```bash
jims@dockeropolis:~$ helm repo add deis https://charts.deis.com/workflow
"deis" has been added to your repositories
```

Finally install the Deis Workflow.  Note that this will take some time.  Using `kubectl` the
progress of bring up can be monitored.

```bash
jims@dockeropolis:~$ helm install deis/workflow --namespace deis
Fetched deis/workflow to workflow-v2.9.0.tgz
NAME: solitary-puma
LAST DEPLOYED: Wed Dec  7 11:17:07 2016
NAMESPACE: deis
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                    TYPE      DATA      AGE
objectstorage-keyfile   Opaque    2         2s
minio-user   Opaque    2         2s
deis-router-dhparam   Opaque    1         2s

==> v1/ConfigMap
NAME                   DATA      AGE
dockerbuilder-config   2         2s
slugbuilder-config   2         2s
slugrunner-config   1         2s

==> v1/ServiceAccount
NAME          SECRETS   AGE
deis-logger   1         2s
deis-controller   1         2s
deis-registry   1         2s
deis-database   1         2s
deis-workflow-manager   1         2s
deis-monitor-telegraf   1         2s
deis-nsqd   1         2s
deis-builder   1         2s
deis-minio   1         2s
deis-router   1         2s
deis-logger-fluentd   1         2s

==> v1/Service
NAME           CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
deis-builder   10.0.35.53   <none>        2222/TCP   2s
deis-database   10.0.138.202   <none>    5432/TCP   2s
deis-router   10.0.214.177   <pending>   80/TCP,443/TCP,2222/TCP,9090/TCP   2s
deis-registry   10.0.136.56   <none>    80/TCP    2s
deis-monitor-grafana   10.0.126.143   <none>    80/TCP    2s
deis-monitor-influxapi   10.0.113.151   <none>    80/TCP    2s
deis-logger-redis   10.0.182.96   <none>    6379/TCP   2s
deis-monitor-influxui   10.0.14.235   <none>    80/TCP    2s
deis-controller   10.0.13.51   <none>    80/TCP    1s
deis-minio   10.0.129.190   <none>    9000/TCP   1s
deis-workflow-manager   10.0.123.151   <none>    80/TCP    1s
deis-logger   10.0.49.27   <none>    80/TCP    1s
deis-nsqd   10.0.186.8   <none>    4151/TCP,4150/TCP   1s

==> extensions/Deployment
NAME                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deis-workflow-manager   1         1         1            0           1s
deis-controller   1         1         1         0         1s
deis-router   1         1         1         0         1s
deis-minio   1         1         1         0         1s
deis-monitor-grafana   1         1         1         0         1s
deis-database   1         1         1         0         1s
deis-monitor-influxdb   1         1         1         0         1s
deis-builder   1         1         1         0         1s
deis-logger   1         0         0         0         1s
deis-nsqd   1         0         0         0         1s
deis-logger-redis   1         0         0         0         1s
deis-registry   1         0         0         0         1s

==> extensions/DaemonSet
NAME                    DESIRED   CURRENT   NODE-SELECTOR   AGE
deis-monitor-telegraf   7         7         <none>          1s
deis-logger-fluentd   7         7         <none>    1s
deis-registry-proxy   7         7         <none>    1s
```

By running `kubectl --namespace=deis get pods`, you can monitor the progress.  An interim 
check make look something like:

```bash
jims@dockeropolis:~$ kubectl --namespace=deis get pods
NAME                                     READY     STATUS             RESTARTS   AGE
deis-builder-415246326-idzx7             1/1       Running            2          5m
deis-controller-1590631305-yfpyq         1/1       Running            4          5m
deis-database-510315365-tdgm9            1/1       Running            0          5m
deis-logger-9212198-plxlx                1/1       Running            2          5m
deis-logger-fluentd-6br1y                1/1       Running            0          5m
deis-logger-fluentd-fi18m                1/1       Running            0          5m
deis-logger-fluentd-fu2r4                1/1       Running            0          5m
deis-logger-fluentd-i9n6m                1/1       Running            0          5m
deis-logger-fluentd-y1uf3                1/1       Running            0          5m
deis-logger-redis-663064164-vf5m0        1/1       Running            0          5m
deis-minio-3160338312-xvi2j              1/1       Running            0          5m
deis-monitor-grafana-432364990-qp26g     1/1       Running            0          5m
deis-monitor-influxdb-2729526471-2uorw   1/1       Running            0          5m
deis-monitor-telegraf-3ye0x              0/1       ImagePullBackOff   0          5m
deis-monitor-telegraf-a2ipa              1/1       Running            0          5m
deis-monitor-telegraf-c60vu              1/1       Running            0          5m
deis-monitor-telegraf-kyq4l              1/1       Running            0          5m
deis-monitor-telegraf-lycuq              1/1       Running            0          5m
deis-nsqd-3264449345-5fe48               1/1       Running            0          5m
deis-registry-2182132043-3fbcy           1/1       Running            1          5m
deis-registry-proxy-72ke5                1/1       Running            0          5m
deis-registry-proxy-99p6b                1/1       Running            0          5m
deis-registry-proxy-enci5                1/1       Running            0          5m
deis-registry-proxy-qa1zn                1/1       Running            0          5m
deis-registry-proxy-sdtas                1/1       Running            0          5m
deis-router-2457652422-fxhsn             1/1       Running            0          5m
deis-workflow-manager-2210821749-bqmya   1/1       Running            0          5m
```

When the cluster is finally up and running, the output should resemble:

```bash
jims@dockeropolis:~$ kubectl --namespace=deis get pods
NAME                                     READY     STATUS    RESTARTS   AGE
deis-builder-415246326-idzx7             1/1       Running   2          6m
deis-controller-1590631305-yfpyq         1/1       Running   4          6m
deis-database-510315365-tdgm9            1/1       Running   0          6m
deis-logger-9212198-plxlx                1/1       Running   2          6m
deis-logger-fluentd-6br1y                1/1       Running   0          6m
deis-logger-fluentd-fi18m                1/1       Running   0          6m
deis-logger-fluentd-fu2r4                1/1       Running   0          6m
deis-logger-fluentd-i9n6m                1/1       Running   0          6m
deis-logger-fluentd-y1uf3                1/1       Running   0          6m
deis-logger-redis-663064164-vf5m0        1/1       Running   0          6m
deis-minio-3160338312-xvi2j              1/1       Running   0          6m
deis-monitor-grafana-432364990-qp26g     1/1       Running   0          6m
deis-monitor-influxdb-2729526471-2uorw   1/1       Running   0          6m
deis-monitor-telegraf-3ye0x              1/1       Running   0          6m
deis-monitor-telegraf-a2ipa              1/1       Running   0          6m
deis-monitor-telegraf-c60vu              1/1       Running   0          6m
deis-monitor-telegraf-kyq4l              1/1       Running   0          6m
deis-monitor-telegraf-lycuq              1/1       Running   0          6m
deis-nsqd-3264449345-5fe48               1/1       Running   0          6m
deis-registry-2182132043-3fbcy           1/1       Running   1          6m
deis-registry-proxy-72ke5                1/1       Running   0          6m
deis-registry-proxy-99p6b                1/1       Running   0          6m
deis-registry-proxy-enci5                1/1       Running   0          6m
deis-registry-proxy-qa1zn                1/1       Running   0          6m
deis-registry-proxy-sdtas                1/1       Running   0          6m
deis-router-2457652422-fxhsn             1/1       Running   0          6m
deis-workflow-manager-2210821749-bqmya   1/1       Running   0          6m
```

At this point, we have Deis Workflow running on Azure.

# Next Steps

In a later tutorial deploying applications and monitoring and other topics will be covered.
