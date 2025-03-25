# checkpoint6
This has the arm template and  parameters file

{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "adminUsername": {
      "type": "string",
      "metadata": { "description": "Admin username for the VM." }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": { "description": "Admin password for the VM." }
    },
    "vmSize": {
      "type": "string",
      "defaultValue": "Standard_B1s",
      "metadata": { "description": "Size of the virtual machine." }
    },
    "allowedSSHSourceIP": {
      "type": "string",
      "defaultValue": "192.168.2.50",
      "metadata": { "description": "Allowed source IP address for SSH access." }
    }
  },
  "variables": {
    "location": "[resourceGroup().location]",
    "vnetName": "Student-1569878-vnet",
    "vnetAddressPrefix": "10.22.39.0/24",
    "subnetName": "Virtual-Desktop-Client",
    "subnetPrefix": "10.22.39.0/24",
    "publicIPName": "myPublicIP",
    "nicName": "myNIC",
    "nsgName": "myNSG",
    "vmName": "mySecureVM",
    "osDiskName": "myOSDisk",
    "imagePublisher": "Canonical",
    "imageOffer": "UbuntuServer",
    "imageSku": "18.04-LTS",
    "customData": "[base64(concat('#!/bin/bash\\n','apt-get update -y\\n','apt-get install -y nginx\\n','systemctl enable nginx\\n','systemctl start nginx\\n'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2020-06-01",
      "name": "[variables('vnetName')]",
      "location": "[variables('location')]",
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "[variables('vnetAddressPrefix')]"
          ]
        },
        "subnets": [
          {
            "name": "[variables('subnetName')]",
            "properties": {
              "addressPrefix": "[variables('subnetPrefix')]",
              "networkSecurityGroup": {
                "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('nsgName'))]"
              }
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/networkSecurityGroups",
      "apiVersion": "2020-06-01",
      "name": "[variables('nsgName')]",
      "location": "[variables('location')]",
      "properties": {
        "securityRules": [
          {
            "name": "Allow-HTTP",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "80",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 100,
              "direction": "Inbound"
            }
          },
          {
            "name": "Allow-HTTPS",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "443",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 110,
              "direction": "Inbound"
            }
          },
          {
            "name": "Allow-SSH",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "sourceAddressPrefix": "[parameters('allowedSSHSourceIP')]",
              "destinationPortRange": "22",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 120,
              "direction": "Inbound"
            }
          },
          {
            "name": "Allow-HTTP-Outbound",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "80",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 130,
              "direction": "Outbound"
            }
          },
          {
            "name": "Allow-HTTPS-Outbound",
            "properties": {
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "443",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 140,
              "direction": "Outbound"
            }
          },
          {
            "name": "Allow-DNS-Outbound",
            "properties": {
              "protocol": "Udp",
              "sourcePortRange": "*",
              "destinationPortRange": "53",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 150,
              "direction": "Outbound"
            }
          },
          {
            "name": "Deny-All-Outbound",
            "properties": {
              "protocol": "*",
              "sourcePortRange": "*",
              "destinationPortRange": "*",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Deny",
              "priority": 200,
              "direction": "Outbound"
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/publicIPAddresses",
      "apiVersion": "2020-06-01",
      "name": "[variables('publicIPName')]",
      "location": "[variables('location')]",
      "properties": {
        "publicIPAllocationMethod": "Static"
      }
    },
    {
      "type": "Microsoft.Network/networkInterfaces",
      "apiVersion": "2020-06-01",
      "name": "[variables('nicName')]",
      "location": "[variables('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/virtualNetworks', variables('vnetName'))]",
        "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName'))]"
      ],
      "properties": {
        "ipConfigurations": [
          {
            "name": "ipconfig1",
            "properties": {
              "subnet": {
                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('vnetName'), variables('subnetName'))]"
              },
              "privateIPAllocationMethod": "Dynamic",
              "publicIPAddress": {
                "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName'))]"
              }
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines",
      "apiVersion": "2021-03-01",
      "name": "[variables('vmName')]",
      "location": "[variables('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
      ],
      "properties": {
        "hardwareProfile": {
          "vmSize": "[parameters('vmSize')]"
        },
        "osProfile": {
          "computerName": "[variables('vmName')]",
          "adminUsername": "[parameters('adminUsername')]",
          "adminPassword": "[parameters('adminPassword')]",
          "customData": "[variables('customData')]"
        },
        "storageProfile": {
          "imageReference": {
            "publisher": "[variables('imagePublisher')]",
            "offer": "[variables('imageOffer')]",
            "sku": "[variables('imageSku')]",
            "version": "latest"
          },
          "osDisk": {
            "name": "[variables('osDiskName')]",
            "caching": "ReadWrite",
            "createOption": "FromImage",
            "managedDisk": {
              "storageAccountType": "Standard_LRS"
            }
          }
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
            }
          ]
        }
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "name": "[concat(variables('vmName'), '/install-nginx')]", // installs nginx upon deployment
      "apiVersion": "2021-03-01",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[resourceId('Microsoft.Compute/virtualMachines', variables('vmName'))]"
      ],
      "properties":
      {
        "publisher": "Microsoft.Azure.Extensions",
        "type": "CustomScript",
        "typeHandlerVersion": "2.0",
        "autoUpgradeMinorVersion": true,
        "settings": {
          "fileUris": [],
          "commandToExecute": "sudo apt update && sudo apt install -y nginx && sudo systemctl enable nginx && sudo systemctl start nginx"
        }
      }
    }
  ],
  "outputs": {
    "publicIPAddress": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName')), '2020-06-01', 'Full').properties.ipAddress]"
    },
    "webURL": {
      "type": "string",
      "value": "[concat('http://', reference(resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName')), '2020-06-01', 'Full').properties.ipAddress)]"
    }
  }
}






Parameters File:
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
      "adminUsername": {
        "value": "huzaifa"
      },
      "adminPassword": {
        "value": "P@ssw0rd1234"
      },
      "vmSize": {
        "value": "Standard_B2s"
      },
      "allowedSSHSourceIP": {
        "value": "50.100.227.246"
      }
    }
  }
 
