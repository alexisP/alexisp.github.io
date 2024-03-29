---
layout: post
title:  "Expose your Kubernetes application securely and privately to your customers/partners through Private Link Service"
author: alexisplantin
comments: false
categories: [ Azure, Kubernetes ]
image: assets/images/aks-and-pls/architecture-2.png
---
This article aims to demonstrate with a quick example how easy it is to expose an AKS application to consumers outside of your organization through the use of Private Link Service and its new integration in AKS. Note that it was already possible to configure a Private Link Service by yourself on top of a Standard Load Balancer generated by AKS, but it was not really integrated, and you had to do some extra work by yourself. Now, exposing a service through Private Link Service in AKS is a matter of a couple of annotations only!

- [What is a private endpoint?](#what-is-a-private-endpoint)
- [What is Private Link Service?](#what-is-private-link-service)
- [AKS and Private Link service integration](#aks-and-private-link-service-integration)
  - [Expose our application to internal networks](#expose-our-application-to-internal-networks)
  - [Expose our application privately through Private Link Service integration](#expose-our-application-privately-through-private-link-service-integration)
    - [Application is unreachable from other non-connected networks](#application-is-unreachable-from-other-non-connected-networks)
    - [Annotate our Service](#annotate-our-service)
    - [Create the Private endpoint](#create-the-private-endpoint)
    - [Approve the private endpoint connection](#approve-the-private-endpoint-connection)
    - [Verify the connection](#verify-the-connection)
- [Useful links](#useful-links)


What is a private endpoint?
---------------------------

In Azure, a private endpoint can be assimilated to a network interface that you deploy on your own virtual network and which points to a remote service hosted on the Azure platform. It is one of the standard and most used ways to establish private connectivity to Azure PaaS services (Storage accounts, SQL Databases, Redis caches...). It is available on mostly all Azure services nowadays, but it could also be used to point to other services hosted in Azure outside of your virtual networks through the use of Private Link Service.

What is Private Link Service?
-------------------------------

Private Link Service is a mechanism that lets you propose to your customers or partners to consume your own services hosted on your virtual networks through private endpoints created on their private virtual networks. This exposition is possible as soon as the service is accessible through a Standard Load Balancer. Here is a schema (from Azure documentation) which illustrates the principle.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/private-link-service-workflow.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

As we can see in the schema above, as soon as the application is exposed behind a Standard Load Balancer:
- You create a Private Link Service instance on top of the Standard Load Balancer.
- You communicate to organizations you would like to allow communication with the Private Link Service resourceId or alias.
- They create the private endpoint associated with your Private Link Service.
- You approve the private endpoint connection (there are also possibilities around visibility of your private link service and auto-approval, but it is out of scope for this article).

More information can be found on Microsoft's official documentations:
- [Private endpoints](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
- [Private Link Service](https://learn.microsoft.com/en-us/azure/private-link/private-link-service-overview)

AKS and Private Link service integration
----------------------------------------

To expose applications and services outside of an AKS cluster, several options are available:
- Ingress controllers (internal and external)
- Application Gateways using AGIC or a modern way through the new Application Gateway for Containers
- Directly through Load Balancers (once again, these LBs could be private or public)

We will focus on the last option and especially on exposing our applications through a private load balancer, then exposed with a Private Link Service. Here is basically what we would like to set up.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/architecture-target.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

To implement it, Microsoft recently released a new AKS feature called Private Link Service integration.

### Expose our application to internal networks

As you probably know, to create a load balancer exposition from within an AKS cluster, we simply need to create a Service object of type LoadBalancer, and AKS automatically creates the appropriate load balancer and load balancing rules. To illustrate it, here is a Pod and a Service associated with this pod:

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <a style="box-shadow:none;" href="https://gist.github.com/alexisP/189da9404b84a17d85909d391d7e5577">
        <img title="Click to access to the source" src="../assets/images/aks-and-pls/sample-app.png"/>
    </a>
  </div>
  <div class="col-md-1"></div>
</section>

Our sample-app application is now available from outside the AKS cluster (but only on connected Vnets as we defined our load balancer as internal. No external customers or partners outside of my tenant can access it yet).

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/architecture-1.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/sample-app-internal.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

### Expose our application privately through Private Link Service integration

#### Application is unreachable from other non-connected networks

Now, let's try to reach the same application but from a spoke which is not connected to our AKS cluster (it is even in another tenant). As expected, we cannot reach the application as no network links are in place.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/sample-app-no-network.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

#### Annotate our Service

To make our application available through Private Link Service, it is as easy as annotating our Service with specific annotations. So, let’s annotate our existing Service to make it exposable and consumable through a Private Link Service. The only required annotation is the `azure-pls-create` annotation which indicates that this service should be exposed using Private Link Service. I added an additional one just to specify the name of my PLS instance using `azure-pls-name` and that's all we need to do to make it exposed. Straightforward!

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <a style="box-shadow:none;" href="https://gist.github.com/alexisP/6750dbc2b08dfed5a358a8fa0411c4ed">
        <img title="Click to access to the source" src="../assets/images/aks-and-pls/sample-app-pls-yml.png"/>
    </a>
  </div>
  <div class="col-md-1"></div>
</section>

After a couple of minutes, the setup is complete and we can find our Private Link service and an associated NIC within the AKS managed resource group (as a reminder, this resource group can be found on the **Properties** tab of your AKS cluster). We then need to get the resourceId of the Private link service, which is required for the next step.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/sample-app-pls.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

#### Create the Private endpoint

On tenant b, we will now create a private endpoint pointing to our Private link service to make our private connectivity to our service live. We reference the resourceId we got from the previous step and we define our Vnet and subnet where our private endpoint will stand.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/create-pe-1.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/create-pe-2.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

Once created, the private endpoint will be in a Pending connection state because the Private Link service owner needs to validate it.

#### Approve the private endpoint connection

On the Private Link service instance side, we can see that we now have a private endpoint connection waiting for our approval, so we just approve it to authorize our customer/partner to reach our service privately through a private endpoint on their network.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/create-pe-4.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

#### Verify the connection

Now, on tenant b, the private endpoint connection status is approved and we can get the private IP of the NIC as shown in the screenshot below.

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/create-pe-5.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

Let's try to curl our service from a VM hosted in the tenant b spoke01 Vnet

<section class="row">
  <div class="col-md-1"></div>
  <div class="col-md-10">
    <img src="../assets/images/aks-and-pls/sample-app-pe.png"/>
  </div>
  <div class="col-md-1"></div>
</section>

It works perfectly fine!

As demonstrated in this article, from a service provider’s point of view, it is really easy now to expose a service privately to consumers or partners using this Private Link Service integration. We made a really simple and short demonstration but if you want to dig more into it, many more options are available on the service (TCP proxy support, FQDNs…) and here is the link to the documentation: [Private Link Service integration](https://cloud-provider-azure.sigs.k8s.io/topics/pls-integration/)

Useful links
------------

- [Private endpoints](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
- [Private Link Service](https://learn.microsoft.com/en-us/azure/private-link/private-link-service-overview)
- [Private Link Service integration](https://cloud-provider-azure.sigs.k8s.io/topics/pls-integration/)