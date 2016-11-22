# Installing Deis on Azure

With the inclusion of Kubernetes as part of the [Azure Container Service](https://azure.microsoft.com/en-us/services/container-service/), 
the deployment of Deis, using the [Deis Workflow](https://github.com/deis/workflow) onto Azure has become 
significantly easier.  The basic steps to deploy Deis into Azure are as follows:

  - Install the Azure CLI
  - Log in to your Azure Account
  - Create a Service Principal (optional)
  - Create the Kubernetes Cluster using the CLI tools
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
second is through the open-sources [acs-engine](https://github.com/Azure/acs-engine) which 
allows you to customize the templates generated.

The standard deployment that is generated (as well as additional instructions for using acs-engine for
Kubernetes) can be found [here](https://github.com/Azure/acs-engine/blob/master/docs/kubernetes.md).

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

If you are specifying a Service Principal, then you will also need the following values.  (Note,
from above, if you don't specify a Service Principal the creation of the Kubernetes Cluster will
create one for you.)

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

Using the above, you run one of the following two commands depending on if you are specifying
the Service Principal or you are not.

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

## Install and Configure kubectl

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
v2.8.0
```

Install Helm Classic:

```bash
jims@dockeropolis:~$ curl -sSL https://get.helm.sh | bash
Downloading helmc-latest-linux-amd64 from Google Cloud Storage...

helmc is now available in your current directory.

To learn more about helm classic, execute:

    $ ./helmc

jims@dockeropolis:~$ mv helmc ~/bin/
jims@dockeropolis:~$ helmc --version
helmc version 0.8.1+a9c55cf
```

### Install Deis Workflow on the Kubernetes Cluster

Since there is already an existing Kubernetes cluster, the path for installing Deis Workflow
follows the [Vagrant procress](https://deis.com/docs/workflow/quickstart/provider/vagrant/install-vagrant/).

First step is to make sure `helmc` finds the config file for `kubectl`.  **Note** - if the terminal 
session was logged out of or restarted or did anything to disrupt the value of `KUBECONFIG` configured
above, the appropriate steps will be needed to ensure it is configured correctly.

```bash
jims@dockeropolis:~$ helmc target
Kubernetes master is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com
Heapster is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/heapster
KubeDNS is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at https://k8sanddeis-k8s-masters.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Looks a lot like the `kubectl cluster-info` call above.

Next step is to add the Deis chart repository:

```bash
jims@dockeropolis:~$ helmc repo add deis https://github.com/deis/charts
---> Checking things locally...
---> Creating /home/jims/.helmc/config.yaml
---> Everything looks good! Happy helming!
---> Continuing onwards and upwards!
---> Cloning into '/home/jims/.helmc/cache/deis'...
---> Hooray! Successfully added the repo.
```

Next retrieve the current Deis Workflow:

```bash
jims@dockeropolis:~$ helmc fetch deis/workflow-v2.8.0 
---> Fetched chart into workspace /home/jims/.helmc/workspace/charts/workflow-v2.8.0
---> Done
```

Create the Deis Workflow manifest for Kubernetes:

```bash
jims@dockeropolis:~$ helmc generate -x manifests workflow-v2.8.0
---> Ran 17 generators.
```

Finally install the Deis Workflow.  Note that this will take some time.  Using `kubectl` the
progress of bring up can be monitored.

```bash
jims@dockeropolis:~$ helmc install workflow-v2.8.0
---> Running `kubectl create -f` ...
namespace "deis" created

secret "builder-ssh-private-keys" created

secret "builder-key-auth" created

secret "django-secret-key" created

secret "database-creds" created

secret "logger-redis-creds" created

secret "objectstorage-keyfile" created

secret "deis-router-dhparam" created

serviceaccount "deis-builder" created

serviceaccount "deis-controller" created

serviceaccount "deis-database" created

serviceaccount "deis-logger-fluentd" created

serviceaccount "deis-logger" created

serviceaccount "deis-minio" created

serviceaccount "deis-monitor-telegraf" created

serviceaccount "deis-nsqd" created

serviceaccount "deis-registry" created

serviceaccount "deis-router" created

serviceaccount "deis-workflow-manager" created

service "deis-builder" created

service "deis-controller" created

service "deis-database" created

service "deis-logger-redis" created

service "deis-logger" created

service "deis-minio" created

service "deis-monitor-grafana" created

service "deis-monitor-influxapi" created

service "deis-monitor-influxui" created

service "deis-nsqd" created

service "deis-registry" created

service "deis-router" created

service "deis-workflow-manager" created

replicationcontroller "deis-builder" created

replicationcontroller "deis-controller" created

replicationcontroller "deis-registry" created

replicationcontroller "deis-router" created

replicationcontroller "deis-workflow-manager" created

deployment "deis-builder" created

deployment "deis-controller" created

deployment "deis-database" created

deployment "deis-logger" created

deployment "deis-logger-redis" created

deployment "deis-minio" created

deployment "deis-monitor-grafana" created

deployment "deis-monitor-influxdb" created

deployment "deis-nsqd" created

deployment "deis-registry" created

deployment "deis-router" created

deployment "deis-workflow-manager" created

daemonset "deis-logger-fluentd" created

daemonset "deis-monitor-telegraf" created

daemonset "deis-registry-proxy" created

---> Done
========================================
# Workflow v2.8.0

Please report any issues you find in testing Workflow to the appropriate GitHub repository:
- builder: https://github.com/deis/builder
- chart: https://github.com/deis/charts
- controller: https://github.com/deis/controller
- database: https://github.com/deis/postgres
- fluentd: https://github.com/deis/fluentd
- helm classic: https://github.com/helm/helm-classic
- logger: https://github.com/deis/logger
- minio: https://github.com/deis/minio
- monitor: https://github.com/deis/monitor
- nsq: https://github.com/deis/nsq
- registry: https://github.com/deis/registry
- router: https://github.com/deis/router
- workflow manager: https://github.com/deis/workflow-manager
========================================
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