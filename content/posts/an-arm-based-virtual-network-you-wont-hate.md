---
title: An ARM based Virtual Network you won't hate!
date: 2014-03-10
categories: ["Azure","ARM","Virtual Network"]
---

Creating an Azure Virtual Network is one of the most imporant, but complex tasks you may be faced with when defining an Azure environment, whether for a pet project, or a corporate engagement.

Whilst there are plenty of examples out there on the resources required for a network, this post outlines the approach and structure I have found best balances simplicity of maintenance with extensibility while keeping the ARM templates terse and easy to understand.

The accompanying sample ARM templates are available at [https://github.com/ross-mcdermott/azure-network-with-arm](https://github.com/ross-mcdermott/azure-network-with-arm).

## What are out goals?

There are a few desired attributes which this approach has been designed to cater for:

- Easy to mange the VNET over time, without needing to be an ARM master.
- Simple to add new Subnets and related NSG / UDR configuration to the VNET.
- Supports the general [ARM best practices](https://blogs.msdn.microsoft.com/mvpawardprogram/2018/05/01/azure-resource-manager/) where possible / practical.
- Conform to 'infrastructure as code' based approches to enable automated deployment
- Don't make the people that support our network hate us...remember..it might be you!

## Embracing convention over configuration

One of the biggest benefits to this approach comes from embracing convention over configuration around the creation of subnets and their related 'best practice' resources (NSGs and UDRs). Following the convention based approach also provides guard rails for subsequent contributors to be guided by and ensures the templates, and therefore the Resource Group they are deployed to, evolve in a familiar, and predictable way.

## Template Structure

Since we are using some convetions within the template, the actual structure of the template is quite simple. The below shows the logical structure of the sample templates:

**--- diagram here ---**

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
        "additionalNsgParameters": {},
        "additionalUdrParameters": {}
    },
    {
        "name": "mysubnet2",
        "addressPrefix": "[variables('subnet-address-prefixes').mysubnet2Prefix]",
        "additionalNsgParameters": {},
        "additionalUdrParameters": {}
    },
    ...
    {
        "name": "[variables('const_GatewaySubnet')]",
        "addressPrefix": "[variables('subnet-address-prefixes').GatewaySubnetPrefix]",
        "additionalNsgParameters": {},
        "additionalUdrParameters": {}
    }
],
"const_GatewaySubnet": "GatewaySubnet"
...
```

The properties are defined as:

| Property                | Type   | Purpose                                                                                                                                                                                                                                     |
| ----------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name                    | string | The name of the subnet to be created                                                                                                                                                                                                        |
| addressPrexix           | string | This is mapped to the subnet when being created                                                                                                                                                                                             |
| additionalNsgParameters | object | The merging (via ARM [union](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-functions-array#union)) of **subnet-address-prefixes** and this property are provided to the subnets NSG nested template |
| additionalUdrParameters | object | The merging (via ARM [union](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-functions-array#union)) of **subnet-address-prefixes** and this property are provided to the subnets UDR nested template |

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

### Subnet UDR Creation: Looping nested templates

The creation of each of the UDR required is achived through the looping of a set of nested templates. This approach ensures that for every subnet there is a UDR resource defined.

To enable a convention based approach, the creation expects all subnet UDR definitons to take two parameters as shown below.

```json
...
    "parameters": {
        "resource-name": {
            "value": "[concat(variables('subnets')[copyIndex('subnet-udr-dependency-loop')].name,'-udr')]"
        },
        "additional-parameters": {
            "value": "[union(variables('subnet-address-prefixes'),variables('subnets')[copyIndex('subnet-udr-dependency-loop')].additionalUdrParameters)]"
        }
    }
...
```

The additional parameters provided is the union of the **subnet-address-prefixes** object and the **additionalNsgParameters** object defined for the subnet. This approach enables each subnet to define any additonal parameters that should be passed to it without affecting the convention based paramter contract.

### Subnet NSG Creation: Looping nested templates

The creation of the NSG resources follows the same convetion as the UDR resources and calls nested templates, the minor logical difference is that the NSG is not created for the SubnetGateway, and excluded with a condition check on the loop.

```json
{
    "condition": "[not(equals(variables('subnets')[copyIndex('subnet-nsg-dependency-loop')].name,variables('const_GatewaySubnet')))]",
    ...
    "copy": {
                "name": "subnet-nsg-dependency-loop",
                "count": "[length(variables('subnets'))]"
            }
    ...
}
```


## Wrapping up...

Hopefully from the above, the benefits of a convention based approach whilst utlising nested templates has ticked a few boxes for you, and the simplicity to create a new subnet stright forward, all we need to do is:

#### 1. Add new subnet prefix

Add a new entry to the **subnet-address-prefixes** variable

```json
   "mysubnet3Prefix": "[concat(parameters('vnet-prefix'), '.3.0/24')]",
```

#### 2. Define the subnet

Add a new entry to the **subnets** variable

```json
   {
        "name": "mysubnet3",
        "addressPrefix": "[variables('subnet-address-prefixes').mysubnet3Prefix]",
        "additionalNsgParameters": {},
        "additionalUdrParameters": {}
    },
```

#### 3. Add nested templates for the subnet

Add new templates for the subnet, for example:

- **nestedtemplates/mysubnet3-nsg.json** - the NSG configuration (ensure filename is lower case)
- **nestedtemplates/mysubnet3-udr.json** - the UDR configuration (ensure filename is lower case)

#### 4. Deploy the network

Deploy the network via you favourite ARM execution method (template = **azuredploy.json**).
