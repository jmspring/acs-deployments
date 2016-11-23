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
  - Scale the Application

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

