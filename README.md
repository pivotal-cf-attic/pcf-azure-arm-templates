# PCF Azure Resource Manager (ARM) Templates

This repo contains ARM templates that help operators deploy Ops Manager Director for Pivotal Cloud Foundry (PCF). 

For more information on installing Pivotal Cloud Foundry, see the [Launching an Ops Manager Director Instance with an ARM Template](https://docs.pivotal.io/pivotalcf/customizing/azure-arm-template.html) topic.

## Template Information

### Parameters

```json
{
    // this is a tag for each resource created, so if this is deployed as part of an existing resource group, it is easy to sort the PCF resources from the rest.
    "Environment": {
      "value": "dev"
    },
    // the Azure region to deploy this in.
    "Location": {
      "value": "westus"
    },
    // The storage account created for the OpsManager VHD
    "OpsManVHDStorageAccount": {
      "value": ""
    },
    // The SSH public key for the PCF Administrator. This key is used to SSH into the OpsManager VM.
    "AdminSSHKey": {
      "value": ""
    },
    // The username of the Administrator for SSH access.
    "AdminUserName": {
      "value": "ubuntu"
    },
    // The storage account container for the OpsManager VM.
    "BlobStorageContainer": {
      "value": "opsman-image"
    }
}
```

## Resources

### Storage Account

**Documentation Reference:** https://docs.microsoft.com/en-us/azure/templates/microsoft.storage/storageaccounts

This is for the Azure Blob Storage configuration, which can be used as a target for the ERT Blob Store.

```json
{
      "type": "Microsoft.Storage/storageAccounts",
      "name": "[concat('pcfblobs', uniqueString(subscription().id))]",
      "apiVersion": "2017-06-01",
      "location": "[parameters('location')]",
      "tags": {
        "Environment": "[parameters('environment')]"
      },
      "kind": "Storage",
      "sku": {
        // Standard LRS has moderate performance with PCF.
        "name": "Standard_LRS"
      }
    }
```

### Network Security Groups

#### Allow Web and SSH

**Documentation Reference:** https://docs.microsoft.com/en-us/azure/templates/microsoft.network/networksecuritygroups/securityrules

This security group allows both web-based and SSH-based traffic through to the Management subnet. It can be use for more subnets, however, it's primary purpose is for the OpsManager VM. The `.properties.securityRules[].properties.destinationAddressPrefix` values can be locked down further with this format: `1.1.1.1/32`.

```json
{
  "name": "AllowWebAndSSH",
  "type": "Microsoft.Network/networkSecurityGroups",
  "apiVersion": "2017-03-01",
  "location": "[parameters('location')]",
  "tags": {
    "Environment": "[parameters('environment')]"
  },
  "properties": {
    "securityRules": [
      {
        "properties": {
          "description": "Allow Inbound HTTP",
          "protocol": "TCP",
          "sourcePortRange": "*",
          "destinationPortRange": "80",
          "sourceAddressPrefix": "*",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 1100,
          "direction": "Inbound"
        },
        "name": "Allow-HTTP"
      },
      {
        "properties": {
          "description": "Allow Inbound HTTPS",
          "protocol": "TCP",
          "sourcePortRange": "*",
          "destinationPortRange": "443",
          "sourceAddressPrefix": "*",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 1000,
          "direction": "Inbound"
        },
        "name": "Allow-HTTPS"
      },
      {
        "properties": {
          "description": "Allow Inbound SSH",
          "protocol": "TCP",
          "sourcePortRange": "22",
          "destinationPortRange": "22",
          "sourceAddressPrefix": "*",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 1300,
          "direction": "Inbound"
        },
        "name": "Allow-SSH"
      }
    ]
  }
}
```

#### Allow Web

**Documentation Reference:** https://docs.microsoft.com/en-us/azure/templates/microsoft.network/networksecuritygroups/securityrules

This security group allows web-based traffic. The `.properties.securityRules[].properties.destinationAddressPrefix` values can be locked down further with this format: `1.1.1.1/32`.

```json
{
  "name": "AllowWeb",
  "type": "Microsoft.Network/networkSecurityGroups",
  "apiVersion": "2017-03-01",
  "location": "[parameters('location')]",
  "tags": {
    "Environment": "[parameters('environment')]"
  },
  "properties": {
    "securityRules": [
      {
        "properties": {
          "description": "Allow Inbound HTTP",
          "protocol": "TCP",
          "sourcePortRange": "*",
          "destinationPortRange": "80",
          "sourceAddressPrefix": "*",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 1100,
          "direction": "Inbound"
        },
        "name": "Allow-HTTP"
      },
      {
        "properties": {
          "description": "Allow Inbound HTTPS",
          "protocol": "TCP",
          "sourcePortRange": "*",
          "destinationPortRange": "443",
          "sourceAddressPrefix": "*",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 1000,
          "direction": "Inbound"
        },
        "name": "Allow-HTTPS"
      }
    ]
  }
}
```

### Virtual Network

**Documentation Reference:** https://docs.microsoft.com/en-us/azure/templates/microsoft.network/virtualnetworks

This is the virtual network configuration for PCF. It contains one top-level class A network: `10.0.0.0/16`. There are three subnets: Management, Services, and Deployment. Each subnet is a `/22` network, aligning with the recommended architecture for PCF. The Management network is designed for OpsManager and other various artifacts, such as jumpboxes, Concourse, etc. The Services network is designed for service-focused tiles, such as the Azure Service Broker, Redis, Spring Cloud Services, etc. The Deployment network is meant for ERT, where your applications will live and run.

```json
{
  "name": "PCF",
  "type": "Microsoft.Network/virtualNetworks",
  "apiVersion": "2017-03-01",
  "location": "[parameters('Location')]",
  "tags": {
    "Environment": "[parameters('environment')]"
  },
  "dependsOn": [
    "[concat('Microsoft.Network/networkSecurityGroups', '/', 'AllowWebAndSSH')]"
  ],
  "properties": {
    "addressSpace": {
      "addressPrefixes": [
        "10.0.0.0/16"
      ]
    },
    "subnets": [
      {
        "name": "Management",
        "properties": {
          "addressPrefix": "10.0.4.0/22",
          "networkSecurityGroup": {
            "id": "[resourceId('Microsoft.Network/networkSecurityGroups', 'AllowWeb')]"
          }
        }
      },
      {
        "name": "Services",
        "properties": {
          "addressPrefix": "10.0.8.0/22",
          "networkSecurityGroup": {
            "id": "[resourceId('Microsoft.Network/networkSecurityGroups', 'AllowWeb')]"
          }
        }
      },
      {
        "name": "Deployment",
        "properties": {
          "addressPrefix": "10.0.12.0/22",
          "networkSecurityGroup": {
            "id": "[resourceId('Microsoft.Network/networkSecurityGroups', 'AllowWeb')]"
          }
        }
      }
    ]
  }
}
```
