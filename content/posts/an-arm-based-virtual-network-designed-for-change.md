---
title: An ARM based Virtual Network designed for change
date: 2019-01-24
categories: ["Azure Resource Manager (ARM)"]
tags: ["Azure","Best Practice","Virtual Network", "ARM"]
---

Creating an ARM template designed for change can be hard. Whilst there are plenty of examples out there on creating basic virtual networks with ARM templates, they generally are not structured to support the ongoing change over time we need. This post outlines an approach and structure that balances simplicity and extensibility while keeping the ARM templates terse and easy to maintain.
<!--more-->

The accompanying sample ARM templates are available on [GitHub](https://github.com/ross-mcdermott/azure-network-with-arm). Other typical VNET resources such as VPN configuration can be easily added to this sample, but have been ommitted to focus on the key concepts.

## What are out goals?

There are a few desired attributes which this approach has been designed to cater for:

- Easy to mange the VNET and related network resources over time, without needing to be an ARM master.
- Simple to add new Subnets and related NSG / UDR configuration to the VNET.
- Supports the general [ARM best practices](https://blogs.msdn.microsoft.com/mvpawardprogram/2018/05/01/azure-resource-manager/) where possible / practical.
- Conform to 'infrastructure as code' based approches to enable automated deployment (scripted or CI/CD pipeline based)

## Embracing convention over configuration

One of the biggest benefits in the approach proposed comes from embracing convention over configuration around the creation of subnets and their related 'best practice' resources (NSGs and UDRs). Following the convention based approach provides guard rails for subsequent contributors and ensures the templates, and therefore the Resource Group they are deployed to, evolve in a familiar, and predictable way.

## Template Structure

Since we are using some convetions within the template, the actual structure of the template is quite simple. The below shows the logical structure of the sample templates:

![Azure VNET ARM Design](/articles/azure-vnet-arm-templates/design.jpg)


### Variables: Subnet Prefixes

The object variable **subnet-address-prefixes** is defined to capture the prefixes for each subnet, this enables a simple 'at a glance' understanding of the shape of the network, and the address spaces in use, and enables rich IDE validation of the properties. 

```json
...
"subnet-address-prefixes": {
    "mysubnet1Prefix": "[concat(parameters('vnet-prefix'), '.1.0/24')]",
    "mysubnet2Prefix": "[concat(parameters('vnet-prefix'), '.2.0/24')]",
    ...
    "GatewaySubnetPrefix": "[concat(parameters('vnet-prefix'), '.255.224/27')]"
},
...
```

A handy tool to calculate the Gateway prefix can be found [here](https://gallery.technet.microsoft.com/scriptcenter/Address-prefix-calculator-a94b6eed)

### Variables: Subnet Configuration

Within the array variable **subnets**, each subnet is configured with basic information, and referencing the appropriate address prefix as defined within  **subnet-address-prefixes**. A slightly special case exists for the GatewaySubnet to ensure the name can be evaluated as a 'constant' in subsequent logic flows to prevent the creation of an NSG resource for it.

```json
...
"subnets": [
    {
        "name": "mysubnet1",
        "addressPrefix": "[variables('subnet-address-prefixes').mysubnet1Prefix]",
        "additionalParameters": {}
    },
    {
        "name": "mysubnet2",
        "addressPrefix": "[variables('subnet-address-prefixes').mysubnet2Prefix]",
        "additionalParameters": {}
    },
    ...
    {
        "name": "[variables('const_GatewaySubnet')]",
        "addressPrefix": "[variables('subnet-address-prefixes').GatewaySubnetPrefix]",
        "additionalParameters": {}
    }
]
...
```

The properties are defined as:

| Property             | Type   | Purpose                                                                                                            |
| -------------------- | ------ | ------------------------------------------------------------------------------------------------------------------ |
| name                 | string | The name of the subnet to be created                                                                               |
| addressPrexix        | string | This is mapped to the subnet when being created                                                                    |
| additionalParameters | object | Provides a standard (convention) for where to define additonal parameters to pass through when creating UDR / NSGs |


### Variables: Context

Define a common context based object which can be simply passed through to dependent templates, and filled out more as required over time. This enables the standard network based information to be provided to subnet based NSGs etc in a standard way.

```json
...
"context": {
    "vnetName": "[variables('vnet-resource-name')]",
    "vnetAddressSpace": "[variables('vnet-address-space')]",
    "subnetAddressPrefixes": "[variables('subnet-address-prefixes')]"
}
...
```

### VNET Creation: Looping the Subnets

Within the VNET resource, a copy loop exists at the property level, enabling us to loop through each of the above defined subnets to generate the subnet definitons in the required format for the VNET resource. The VNET has an ARM **dependsOn** for the creation of the UDR and NSG resources.

```json
...
"copy": [
    {
        "name": "subnets",
        "count": "[length(variables('subnets'))]",
        "input": {
            "name": "[variables('subnets')[copyIndex('subnets')].name]",
            "properties": {
                "addressPrefix": "[variables('subnets')[copyIndex('subnets')].addressPrefix]",
                "networkSecurityGroup": "[if(not(equals(variables('subnets')[copyIndex('subnets')].name,variables('const_GatewaySubnet'))), json(concat('{\"id\": \"', resourceId(resourceGroup().name, 'Microsoft.Network/networkSecurityGroups/', concat(variables('subnets')[copyIndex('subnets')].name,'-nsg')), '\"}')),json('null'))]",
                "routeTable": {
                    "id": "[resourceId(resourceGroup().name, 'Microsoft.Network/routeTables/', concat(variables('subnets')[copyIndex('subnets')].name,'-udr'))]"
                }
            }
        }
    }
]
...
```

You will notice some conditions and logic defining the **networkSecurityGroup** property - this is due of desire to keep the template convetion based while the GatewaySubnet is actually a little different in that it is not supported to apply an NSG to it. Unfortunately the expression and logic syntax for ARM is not particarly readable, so in pseudo code, the expression is in effect doing this logic for us:

```js
if(loop[n].name != 'GatewaySubnet'){ 
    subnet.networkSecurityGroup = {
        "id": "<NSG Resouce ID>"
    }
}
else {
    subnet.networkSecurityGroup = null;
}
```

### Subnet Dependency Creation: Looping nested template

For each subnet, the template **nestedtemplate/subnet-dependencies.json** is called as a sub-deployment, passing through the subnet to be created, and the **context** variable used to provide standard information to the template.

```json
...
    "parameters": {
        ...
        "subnet": {
            "value": "[variables('subnets')[copyIndex('subnet-dependency-loop')]]"
        },
        "context": {
            "value": "[variables('context')]"
        }
    }
...
```

This template then in turn calls the specific template required based on the subnet object provided. The naming convention required for the NSG and UDR resource templates for each subnet are **nestedtemplates/subnet-{name}-nsg.json** and **nestedtemplates/subnet-{name}-udr.json** respectively.

## Summary

By using a convention based approach, the top level template can be kept simple, while allowing detailed information to be passed through to the nested templates. The additon and defintion of new subnets, and related NSG / UDR rules can be achived easily within minimal changes while retaining the aspect of the best practice use of ARM templates.


