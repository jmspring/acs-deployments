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