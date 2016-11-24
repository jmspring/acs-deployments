# Deploying an Application to Deis on Azure

This tutorial will walk through deploying an application on to the Deis Workflow that 
was deployed using [this](https://github.com/jmspring/acs-deployments/blob/master/deis_on_azure/deis_on_azure.md)
walk through.

In this tutorial, the [Deis Workflow Quickstart](https://deis.com/docs/workflow/quickstart/) will be 
followed, specifically the [Deploy an App](https://deis.com/docs/workflow/quickstart/deploy-an-app/)
section.  The write up will be catered to nuances particular to Azure, including setting up Azure DNS
for use with Deis.  The area most particular to Azure will be DNS and wildcard record setup.

The steps for deploying an application are:

  - Setup Azure DNS
    - Create the Azure DNS Zone
    - Add a Wildcard A Record
  - Register an Admin User
  - Deploy an Application
  - Change the Application Configuration

## Setup Azure DNS

Deis requires that a wildcard DNS record be setup pointing to the router such that applications
can be dynamically routed.  For this tutorial, Azure DNS will be used.  In the 
[Deis Workflow walkthrough](https://deis.com/docs/workflow/quickstart/deploy-an-app/) <http://nip.io>
is used.  For this tutorial, an actual domain will be used and DNS and records set up accordingly.

The process for setting up Azure DNS is as follows:

  - Determine the Domain Name to use
  - Create an Azure DNS Zone for the Domain Name
  - Update DNS Entries for the Domain Name at the Domain Registrar
  - Create a Wildcard A Record Set in the Azure DNS Zone
  - Test to Verify

If an existing Domain Name is available, make sure it is not configured to an existing host / DNS
provider.  If a new Domain Name is desired, follow the steps necessary for the registrar of choice.
For this tutorial, the Domain Name `plusonetechnology.net` which is registered with <http://joker.com>
will be used.

To create an Azure DNS Zone using the [Azure CLI](https://github.com/Azure/azure-cli), a resource 
group is needed.  This tutorial will continue to use the `resource group` from the 
[Installing Deis on Azure](https://github.com/jmspring/acs-deployments/blob/master/deis_on_azure/deis_on_azure.md)
which was `deisonk8srg`.

```bash
jims@dockeropolis:~$ az network dns zone create --name="plusonetechnology.net" --resource-group="deisonk8srg" 
{
  "etag": "00000002-0000-0000-39e8-9a7bc045d201",
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Network/dnszones/plusonetechnology.net",
  "location": "global",
  "maxNumberOfRecordSets": 5000,
  "name": "plusonetechnology.net",
  "nameServers": [
    "ns1-07.azure-dns.com.",
    "ns2-07.azure-dns.net.",
    "ns3-07.azure-dns.org.",
    "ns4-07.azure-dns.info."
  ],
  "numberOfRecordSets": 2,
  "resourceGroup": "deisonk8srg",
  "tags": {},
  "type": "Microsoft.Network/dnszones"
}
```

In the information returned, note the list of name servers.  In order to delegate DNS resolution for 
the domain choser, the DNS entries for the domain need to be updated at the registrar the domain is 
registered with.  For the domain `plusonetechnology.net`, the registrar is [Joker](http://joker.com).  

Updating the DNS records at Joker for the name will look like:

<img src="https://raw.githubusercontent.com/jmspring/acs-deployments/master/deis_on_azure/joker_dns.png" width="480">

To create the Wildcard A Record in the Azure DNS Zone, the public IP address of the Deis Router IP is needed.
To get that information, use `kubectl`:

```bash
jims@dockeropolis:~$ kubectl --namespace=deis describe svc deis-router
Name:			deis-router
Namespace:		deis
Labels:			heritage=deis
Selector:		app=deis-router
Type:			LoadBalancer
IP:			10.0.155.69
LoadBalancer Ingress:	168.61.71.200
Port:			http	80/TCP
NodePort:		http	30912/TCP
Endpoints:		10.244.4.6:8080
Port:			https	443/TCP
NodePort:		https	32356/TCP
Endpoints:		10.244.4.6:6443
Port:			builder	2222/TCP
NodePort:		builder	31775/TCP
Endpoints:		10.244.4.6:2222
Port:			healthz	9090/TCP
NodePort:		healthz	32570/TCP
Endpoints:		10.244.4.6:9090
Session Affinity:	None
```

From the above, the IP address needed is the `LoadBalancer Ingress`, which is `168.61.71.200`.

Using this value, the Wildcard A Record is in two stages.  First a Record Set is created within
the DNS Zone `plusonetechnology.net`:

```bash
jims@dockeropolis:~$ az network dns record-set create \
> --resource-group="deisonk8srg" \
> --zone-name="plusonetechnology.net" \
> --type="A" \
> --name="*"
{
  "aaaaRecords": null,
  "arecords": [],
  "cnameRecord": null,
  "etag": "c8ad5d09-8e0b-49aa-b531-b1c62b17cbdc",
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Network/dnszones/plusonetechnology.net/A/*",
  "metadata": null,
  "mxRecords": null,
  "name": "*",
  "nsRecords": null,
  "ptrRecords": null,
  "resourceGroup": "deisonk8srg",
  "soaRecord": null,
  "srvRecords": null,
  "ttl": 3600,
  "txtRecords": null,
  "type": "Microsoft.Network/dnszones/A"
}
```

Second, you need to add the Deis `LoadBalancer Ingress` IP address to the record:

```bash
jims@dockeropolis:~$ az network dns record a add \
> --resource-group="deisonk8srg" \
> --zone-name="plusonetechnology.net" \
> --record-set-name="*" \
> --ipv4-address="168.61.71.200"
{
  "aaaaRecords": null,
  "arecords": [
    {
      "ipv4Address": "168.61.71.200"
    }
  ],
  "cnameRecord": null,
  "etag": "c0b891dd-7623-4770-8fac-904eabd4b8b8",
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Network/dnszones/plusonetechnology.net/A/*",
  "metadata": null,
  "mxRecords": null,
  "name": "*",
  "nsRecords": null,
  "ptrRecords": null,
  "resourceGroup": "deisonk8srg",
  "soaRecord": null,
  "srvRecords": null,
  "ttl": 3600,
  "txtRecords": null,
  "type": "Microsoft.Network/dnszones/A"
}
```

To list record set details:

```bash
jims@dockeropolis:~$ az network dns record-set show \
> --resource-group="deisonk8srg" 
> --zone-name="plusonetechnology.net" 
> --type=A 
> --name="*"
{
  "aaaaRecords": null,
  "arecords": [
    {
      "ipv4Address": "168.61.71.200"
    }
  ],
  "cnameRecord": null,
  "etag": "c0b891dd-7623-4770-8fac-904eabd4b8b8",
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/deisonk8srg/providers/Microsoft.Network/dnszones/plusonetechnology.net/A/*",
  "metadata": null,
  "mxRecords": null,
  "name": "*",
  "nsRecords": null,
  "ptrRecords": null,
  "resourceGroup": "deisonk8srg",
  "soaRecord": null,
  "srvRecords": null,
  "ttl": 3600,
  "txtRecords": null,
  "type": "Microsoft.Network/dnszones/A"
}
```

Which looks identical to the output of adding the IPv4 address to the record in the prior step.  To note:
the older Azure-xplat-cli allowed this to be done in one step, why Azure CLI 2.0 Preview requires two 
steps is a curious requirement.  There are issues open around this on [github](https://github.com/Azure/azure-cli/issues/1134).

To verify DNS is working correctly and that multiple domain names resolve to the `LoadBalancer Ingress` IP,
try a couple of calls to `nslookup`:

```bash
jims@dockeropolis:~$ nslookup gonzo.plusonetechnology.net
Server:		10.211.55.1
Address:	10.211.55.1#53

Non-authoritative answer:
Name:	gonzo.plusonetechnology.net
Address: 168.61.71.200

jims@dockeropolis:~$ nslookup rizzo.plusonetechnology.net
Server:		10.211.55.1
Address:	10.211.55.1#53

Non-authoritative answer:
Name:	rizzo.plusonetechnology.net
Address: 168.61.71.200
```

## Register an Admin User

Now that DNS is setup for the Deis Workflow, the next step is to register an admin user with
the Deis Workflow.  The details we will use for the admin account are:

  - username: admin
  - password: <password of choice>
  - email: jaspring@microsoft.com

The command:

```
jims@dockeropolis:~$ deis register http://deis.plusonetechnology.net
username: admin
password: 
password (confirm): 
email: jaspring@microsoft.com
Registered admin
username: admin
password: 
Logged in as admin
Configuration file written to /home/jims/.deis/client.json
```

Upon registering you are prompted for the details (as noted above).  The second request for
username and password is to actually log in do Deis.

With the user created and logged in, an application can now be deployed.

## Deploy an Application

To deploy an application per the Deis Workflow tutorial, there are two steps to follow:

  - Create a new application (more like application name space)
  - Deploy an application into that name space

To create the application:

```bash
jims@dockeropolis:~$ deis create --no-remote
Creating Application... done, created flaxen-lambskin
If you want to add a git remote for this app later, use `deis git:remote -a flaxen-lambskin`
```

Note, you can specify a name, but if you don't, Deis can come up with some fun name options.

Part of what  the `deis create` command actually does is sets up a mapping for incoming requests
to the application, so HTTP requests to (in the case of above) `http://flaxen-lambskin.plusonetechnology.net` 
will get routed to the application deployed to that name space.

Per the Deis Workflow, an application will deployed to the namespace generated, in this case
`flaxen-lambskin`.  To do such, using the Deis Workflow example:

```bash
jims@dockeropolis:~$ deis pull deis/example-go -a flaxen-lambskin
Creating build... done
```

To see the application was deployed, let's make a `curl` call:

```bash
jims@dockeropolis:~$ curl http://flaxen-lambskin.plusonetechnology.net
Powered by Deis
```

## Change the Application Configuration

At this point, we have the Deis Workflow sample application deployed and working on Kubernetes on Azure.  
The next step in the tutorial is to change the configuration:

```bash
jims@dockeropolis:~$ deis config:set POWERED_BY="Docker Images + Kubernetes + Azure" -a flaxen-lambskin
Creating config... done

=== flaxen-lambskin Config
POWERED_BY      Docker Images + Kubernetes + Azure
```

To verify the configuration change:

```bash
jims@dockeropolis:~$ curl flaxen-lambskin.plusonetechnology.net
Powered by Docker Images + Kubernetes + Azure
```

The [Deis Workflow - Deploy an App](https://deis.com/docs/workflow/quickstart/deploy-an-app/) documentation
describes scaling the app, but just builds on our already working situation.

