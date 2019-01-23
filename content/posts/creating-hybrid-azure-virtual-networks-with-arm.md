---
title: Creating Hybrid Azure Virtual Network with ARM templates
date: 2014-03-10
categories: ["Azure","ARM","DevOps","Network"]
---

Creating an Azure Virtual Network is one of the most imporant, but daunting tasks you may be faced with when defining your Azure environment(s) and enabling connectivity back to on premises (or other) network in a hybrid environment.

The creation and management of a Virtual Network is something that aligns very well to use of ARM Templates, however without a clear approach to the problem, the template(s) quickly morph into an unmanagble state after a few changes are rolled out. By following a standardised approach to the problem, common changes such as the additon of new Subnets and changes to UDR / NSG configuration become much simpler.


## What makes networks related templates complex?
Simply put, what makes Virtual Network ARM templates complex (especially for a hybid network) is the number of resources that need to be defined to create a usable, secure network, and the dependencies / relationships between those resources. The following is an example for the resources we need to create for a simple 3 + Gateway subnet network, configured with a VPN connection to another network.

| Resource Type | Quantity | Description |
|------------------|----------------------------|------------------------------|
|Microsoft.Network/virtualNetworks|1|Define the VNET, and the subnets within it|
|Microsoft.Network/virtualNetworkGateways|1| The gateway definiton to specify gateway type, and additionally enable P2S connectivity to the network|
|Microsoft.Network/publicIPAddresses|1| Define the IP Address that the Virtual Network Gateway will be available at |
|Microsoft.Network/localNetworkGateways| 1 | Define IP Address and Address Space(s) for the remote network |
|Microsoft.Network/connections| 1 | Configure the Virtual Network Gateway to connect to the Local Newtork Gatway|
|Microsoft.Network/networkSecurityGroups|Count(Subnets)|Define the NSGs for each subnet|
|Microsoft.Network/routeTables|Count(Subnets) + GatewaySubnet|Define the NSGs for each subnet|
|*Total*|12 Resources|

In additon to the above, limitations on the VNET resource type within ARM means we cannot split the definiton of each subnet into its own template without some side effects we don't want to live with.

## Whats the plan?





## Resource to help plan:
https://gallery.technet.microsoft.com/scriptcenter/Address-prefix-calculator-a94b6eed


