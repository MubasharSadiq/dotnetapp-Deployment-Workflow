# **Project 2 - Azure**

`Mubashar Sadiq`

---

# **Deploying a .NET Application on Azure with ARM Templates, Bastion Host, Reverse Proxy, and CI/CD**

This guide demonstrates the process of provisioning and deploying a .NET application on Azure using ARM templates and setting up a CI/CD pipeline with GitHub Actions. The deployment includes infrastructure provisioning, reverse proxy configuration, bastion host, and automated application deployment.

---

## **Table of Contents**
- [Prerequisites](#prerequisites)
- [Step 1: Building the .NET Web Application](#step-1-building-the-net-web-application)
- [Step 2: Deploying Azure Infrastructure Using ARM Templates](#step-2-deploying-azure-infrastructure-using-arm-templates)
  - [2.1: Prepare the ARM Template](#21-prepare-the-arm-template)
  - [2.2: Prepare the Deployment Script](#22-prepare-the-deployment-script)
  - [2.3: Run the Deployment Script](#23-run-the-deployment-script)
- [Step 3: Configuring the Reverse Proxy](#step-3-configuring-the-reverse-proxy)
- [Step 4: Configuring the Application Server](#step-4-configuring-the-application-server)
- [Step 5: Deploying the Application](#step-5-deploying-the-application)
  - [Setting up the CI Workflow](#setting-up-the-ci-workflow)
  - [Configuring the Deployment Workflow (CD)](#configuring-the-deployment-workflow-cd)
  - [Setting up a Self-Hosted Runner on the VM](#setting-up-a-self-hosted-runner-on-the-vm)
- [Step 6: Verifying the Deployment](#step-6-verifying-the-deployment)
- [Scripts](#scripts)
  - [ARM Template for Infrastructure](#arm-template-for-infrastructure)
  - [Deployment Script](#deployment-script)

---

## **Prerequisites**
- Azure Account
- Azure CLI:
  ```bash
  az login
  ```
- Git and GitHub Account
- .NET 8.0 SDK:
  ```bash
  # Install .NET 8.0 SDK on Ubuntu
  wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  sudo dpkg -i packages-microsoft-prod.deb
  sudo apt-get update
  sudo apt-get install -y dotnet-sdk-8.0
  ```

[Back to top](#table-of-contents)

---

## **Step 1: Building the .NET Web Application**
1. Create a new .NET Web App project in a directory called `dotnetapp`:
   ```bash
   dotnet new webapp -o dotnetapp
   cd dotnetapp
   ```
2. Run the application locally to verify it works:
   ```bash
   dotnet run
   ```
3. Initialize a Git repository and push the code to GitHub:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/yourusername/dotnetapp.git
   git push -u origin main
   ```

[Back to top](#table-of-contents)

---

## **Step 2: Deploying Azure Infrastructure Using ARM Templates**
### 2.1: Prepare the ARM Template
Define the Azure infrastructure in an ARM template named `IaaS-armTemplate.json`. The template should include:
- Virtual Network (VNet) with one subnet
- Network Security Groups (NSGs) for each VM
- Public IP addresses for the Reverse Proxy and Bastion Host
- Network Interfaces (NICs) for each VM
- Virtual Machines:
  - Reverse Proxy VM
  - Application Server VM (dotnetAppVM)
  - Bastion Host VM

[Click here to access the ARM Template](#arm-template-for-infrastructure)

### 2.2: Prepare the Deployment Script
Create a deployment script named `deploy-armTemplate.sh` that does the following:
- Checks for Azure CLI and SSH key
- Creates the resource group if it doesn't exist
- Deploys the ARM template using Azure CLI

[Click here to access the Deployment script](#deployment-script)

### 2.3: Run the Deployment Script
1. Make the script executable:
   ```bash
   chmod +x deploy-armTemplate.sh
   ```
2. Run the script:
   ```bash
   ./deploy-armTemplate.sh
   ```
3. Verify the deployment in the Azure Portal or by running:
   ```bash
   az resource list --resource-group "dotnetAppRG"
   ```

[Back to top](#table-of-contents)

---

## **Step 3: Configuring the Reverse Proxy**
1. SSH into the Bastion Host:
   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_rsa
   bastionHostPublicIP=$(az network public-ip show --resource-group dotnetAppRG --name bastionHostPublicIP --query ipAddress -o tsv)
   ssh -A azureuser@$bastionHostPublicIP
   ```
2. SSH into the Reverse Proxy VM from the Bastion Host:
   ```bash
   reverseProxyPrivateIP=$(az vm list-ip-addresses --resource-group dotnetAppRG --name reverseProxyVM --query "[].virtualMachine.network.privateIpAddresses[0]" -o tsv)
   ssh azureuser@$reverseProxyPrivateIP
   ```
3. Install Nginx:
   ```bash
   sudo apt-get update
   sudo apt-get install -y nginx
   ```
4. Configure Nginx:
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```
   Replace the contents with:
   ```nginx
   server {
     listen 80 default_server;
     location / {
       proxy_pass http://localhost:5000/;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection keep-alive;
       proxy_set_header Host $host;
       proxy_cache_bypass $http_upgrade;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
     }
   }
   ```
5. Restart Nginx:
   ```bash
   sudo systemctl restart nginx
   ```
6. Test the Reverse Proxy:
   ```bash
   reverseProxyPublicIP=$(az network public-ip show --resource-group dotnetAppRG --name reverseProxyPublicIP --query ipAddress -o tsv)
   curl http://$reverseProxyPublicIP
   ```

[Back to top](#table-of-contents)

---

## **Step 4: Configuring the Application Server**
1. SSH into the Application Server VM via the Bastion Host:
   ```bash
   appServerPrivateIP=$(az vm list-ip-addresses --resource-group dotnetAppRG --name dotnetAppVM --query "[].virtualMachine.network.privateIpAddresses[0]" -o tsv)
   ssh azureuser@$appServerPrivateIP
   ```
2. Install the .NET 8.0 Runtime:
   ```bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https
   wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
   sudo dpkg -i packages-microsoft-prod.deb
   sudo apt-get update
   sudo apt-get install -y dotnet-runtime-8.0
   ```
3. Create a systemd service for the application:
   ```bash
   sudo nano /etc/systemd/system/dotnetapp.service
   ```
   Add the following content:
   ```ini
   [Unit]
   Description=ASP.NET Web App running on Ubuntu
   After=network.target
   
   [Service]
   WorkingDirectory=/opt/dotnetapp
   ExecStart=/usr/bin/dotnet /opt/dotnetapp/dotnetapp.dll
   Restart=always
   RestartSec=10
   KillSignal=SIGINT
   SyslogIdentifier=dotnetapp
   User=www-data
   Environment=ASPNETCORE_ENVIRONMENT=Production
   Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
   Environment="ASPNETCORE_URLS=http://*:5000"
   
   [Install]
   WantedBy=multi-user.target
   ```
4. Ensure correct ownership and permissions for the application directory:
   ```bash
   sudo chown -R www-data:www-data /opt/dotnetapp
   sudo chmod -R 755 /opt/dotnetapp
   ```

[Back to top](#table-of-contents)

---

## **Step 5: Deploying the Application**
### Setting up the CI Workflow
1. In the GitHub repo, navigate to the Actions tab and set up a new .NET workflow.
2. Configure the workflow to build and publish the application.

### Configuring the Deployment Workflow (CD)
1. Update the workflow to include a deployment job that downloads the build artifacts and deploys them to the VM.
2. Use a self-hosted runner for the deployment tasks.

### Setting up a Self-Hosted Runner on the VM
1. Follow the GitHub instructions to set up a self-hosted runner on the Application Server VM:
   ```bash
   mkdir actions-runner && cd actions-runner
   curl -o actions-runner-linux-x64-2.304.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.304.0/actions-runner-linux-x64-2.304.0.tar.gz
   tar xzf ./actions-runner-linux-x64-2.304.0.tar.gz
   ./config.sh --url https://github.com/yourusername/dotnetapp --token <your-github-token>
   ./run.sh
   ```
2. Verify the runner is connected and idle.

[Back to top](#table-of-contents)

---

## **Step 6: Verifying the Deployment**
1. Push a change to the `main` branch to trigger the CI/CD pipeline:
   ```bash
   git add .
   git commit -m "Trigger CI/CD"
   git push
   ```
2. Monitor the pipeline to ensure the build and deployment jobs complete successfully.
3. Access the application via the Reverse Proxy server's public IP to verify it is working correctly:
   ```bash
   reverseProxyPublicIP=$(az network public-ip show --resource-group dotnetAppRG --name reverseProxyPublicIP --query ipAddress -o tsv)
   curl http://$reverseProxyPublicIP
   ```

[Back to top](#table-of-contents)

---

## **Scripts**

### **ARM Template for Infrastructure**
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "adminUsername": {
            "type": "string",
            "metadata": {
                "description": "Admin username for the VMs."
            }
        },
        "sshPublicKey": {
            "type": "string",
            "metadata": {
                "description": "SSH public key for VM authentication."
            }
        },
        "vmSize": {
            "type": "string",
            "defaultValue": "Standard_B1ls",
            "metadata": {
                "description": "VM size for the virtual machines."
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]",
            "metadata": {
                "description": "Location for all resources."
            }
        }
    },
    "variables": {
        "vNetName": "dotnetAppVNet",
        "reverseProxyNic": "reverseProxyNic",
        "appServerNic": "dotnetAppServerNic",
        "bastionHostNic": "BastionHostNic",
        "subnetName": "dotnetAppSubnet",
        "reverseProxyPublicIPName": "reverseProxyPublicIP",
        "bastionHostPublicIPName": "bastionHostPublicIP"
    },
    "resources": [
        {
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2020-06-01",
            "name": "[variables('vNetName')]",
            "location": "[parameters('location')]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.0.0.0/16"
                    ]
                },
                "subnets": [
                    {
                        "name": "[variables('subnetName')]",
                        "properties": {
                            "addressPrefix": "10.0.1.0/24"
                        }
                    }
                ]
            }
        },
        {
            "type": "Microsoft.Network/networkSecurityGroups",
            "apiVersion": "2020-06-01",
            "name": "reverseProxyNSG",
            "location": "[parameters('location')]",
            "properties": {
                "securityRules": [
                    {
                        "name": "AllowHTTP",
                        "properties": {
                            "protocol": "Tcp",
                            "sourcePortRange": "*",
                            "destinationPortRange": "80",
                            "sourceAddressPrefix": "*",
                            "destinationAddressPrefix": "*",
                            "access": "Allow",
                            "priority": 1000,
                            "direction": "Inbound"
                        }
                    },
                    {
                        "name": "AllowHTTPS",
                        "properties": {
                            "protocol": "Tcp",
                            "sourcePortRange": "*",
                            "destinationPortRange": "443",
                            "sourceAddressPrefix": "*",
                            "destinationAddressPrefix": "*",
                            "access": "Allow",
                            "priority": 1001,
                            "direction": "Inbound"
                        }
                    },
                    {
                        "name": "AllowSSHFromBastion",
                        "properties": {
                            "protocol": "Tcp",
                            "sourcePortRange": "*",
                            "destinationPortRange": "22",
                            "sourceAddressPrefix": "10.0.1.0/24",
                            "destinationAddressPrefix": "*",
                            "access": "Allow",
                            "priority": 1002,
                            "direction": "Inbound"
                        }
                    }
                ]
            }
        },
        {
            "type": "Microsoft.Network/networkSecurityGroups",
            "apiVersion": "2020-06-01",
            "name": "appServerNSG",
            "location": "[parameters('location')]",
            "properties": {
                "securityRules": [
                    {
                        "name": "AllowHTTPFromReverseProxy",
                        "properties": {
                            "protocol": "Tcp",
                            "sourcePortRange": "*",
                            "destinationPortRange": "80",
                            "sourceAddressPrefix": "10.0.1.0/24",
                            "destinationAddressPrefix": "*",
                            "access": "Allow",
                            "priority": 1000,
                            "direction": "Inbound"
                        }
                    },
                    {
                        "name": "AllowSSHFromBastion",
                        "properties": {
                            "protocol": "Tcp",
                            "sourcePortRange": "*",
                            "destinationPortRange": "22",
                            "sourceAddressPrefix": "10.0.1.0/24",
                            "destinationAddressPrefix": "*",
                            "access": "Allow",
                            "priority": 1001,
                            "direction": "Inbound"
                        }
                    }
                ]
            }
        },
        {
            "type": "Microsoft.Network/networkSecurityGroups",
            "apiVersion": "2020-06-01",
            "name": "bastionHostNSG",
            "location": "[parameters('location')]",
            "properties": {
                "securityRules": [
                    {
                        "name": "AllowSSH",
                        "properties": {
                            "protocol": "Tcp",
                            "sourcePortRange": "*",
                            "destinationPortRange": "22",
                            "sourceAddressPrefix": "*",
                            "destinationAddressPrefix": "*",
                            "access": "Allow",
                            "priority": 1000,
                            "direction": "Inbound"
                        }
                    }
                ]
            }
        },
        {
            "type": "Microsoft.Network/publicIPAddresses",
            "apiVersion": "2020-06-01",
            "name": "[variables('reverseProxyPublicIPName')]",
            "location": "[parameters('location')]",
            "properties": {
                "publicIPAllocationMethod": "Dynamic"
            }
        },
        {
            "type": "Microsoft.Network/publicIPAddresses",
            "apiVersion": "2020-06-01",
            "name": "[variables('bastionHostPublicIPName')]",
            "location": "[parameters('location')]",
            "properties": {
                "publicIPAllocationMethod": "Dynamic"
            }
        },
        {
            "type": "Microsoft.Network/networkInterfaces",
            "apiVersion": "2020-06-01",
            "name": "[variables('reverseProxyNic')]",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/virtualNetworks', variables('vNetName'))]",
                "[resourceId('Microsoft.Network/publicIPAddresses', variables('reverseProxyPublicIPName'))]",
                "[resourceId('Microsoft.Network/networkSecurityGroups', 'reverseProxyNSG')]"
            ],
            "properties": {
                "ipConfigurations": [
                    {
                        "name": "ipconfig1",
                        "properties": {
                            "subnet": {
                                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('vNetName'), variables('subnetName'))]"
                            },
                            "privateIPAllocationMethod": "Dynamic",
                            "publicIPAddress": {
                                "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('reverseProxyPublicIPName'))]"
                            }
                        }
                    }
                ],
                "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', 'reverseProxyNSG')]"
                }
            }
        },
        {
            "type": "Microsoft.Network/networkInterfaces",
            "apiVersion": "2020-06-01",
            "name": "[variables('appServerNic')]",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/virtualNetworks', variables('vNetName'))]",
                "[resourceId('Microsoft.Network/networkSecurityGroups', 'appServerNSG')]"
            ],
            "properties": {
                "ipConfigurations": [
                    {
                        "name": "ipconfig1",
                        "properties": {
                            "subnet": {
                                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('vNetName'), variables('subnetName'))]"
                            },
                            "privateIPAllocationMethod": "Dynamic"
                        }
                    }
                ],
                "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', 'appServerNSG')]"
                }
            }
        },
        {
            "type": "Microsoft.Network/networkInterfaces",
            "apiVersion": "2020-06-01",
            "name": "[variables('bastionHostNic')]",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/virtualNetworks', variables('vNetName'))]",
                "[resourceId('Microsoft.Network/publicIPAddresses', variables('bastionHostPublicIPName'))]",
                "[resourceId('Microsoft.Network/networkSecurityGroups', 'bastionHostNSG')]"
            ],
            "properties": {
                "ipConfigurations": [
                    {
                        "name": "ipconfig1",
                        "properties": {
                            "subnet": {
                                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('vNetName'), variables('subnetName'))]"
                            },
                            "privateIPAllocationMethod": "Dynamic",
                            "publicIPAddress": {
                                "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('bastionHostPublicIPName'))]"
                            }
                        }
                    }
                ],
                "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', 'bastionHostNSG')]"
                }
            }
        },
        {
            "type": "Microsoft.Compute/virtualMachines",
            "apiVersion": "2020-06-01",
            "name": "reverseProxyVM",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/networkInterfaces', variables('reverseProxyNic'))]"
            ],
            "properties": {
                "hardwareProfile": {
                    "vmSize": "[parameters('vmSize')]"
                },
                "osProfile": {
                    "computerName": "reverseProxyVM",
                    "adminUsername": "[parameters('adminUsername')]",
                    "linuxConfiguration": {
                        "disablePasswordAuthentication": true,
                        "ssh": {
                            "publicKeys": [
                                {
                                    "path": "[concat('/home/', parameters('adminUsername'), '/.ssh/authorized_keys')]",
                                    "keyData": "[parameters('sshPublicKey')]"
                                }
                            ]
                        }
                    }
                },
                "networkProfile": {
                    "networkInterfaces": [
                        {
                            "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('reverseProxyNic'))]"
                        }
                    ]
                },
                "storageProfile": {
                    "imageReference": {
                        "publisher": "Canonical",
                        "offer": "0001-com-ubuntu-server-jammy",
                        "sku": "22_04-lts-gen2",
                        "version": "latest"
                    },
                    "osDisk": {
                        "createOption": "FromImage"
                    }
                }
            }
        },
        {
            "type": "Microsoft.Compute/virtualMachines",
            "apiVersion": "2020-06-01",
            "name": "dotnetAppVM",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/networkInterfaces', variables('appServerNic'))]"
            ],
            "properties": {
                "hardwareProfile": {
                    "vmSize": "[parameters('vmSize')]"
                },
                "osProfile": {
                    "computerName": "dotnetAppVM",
                    "adminUsername": "[parameters('adminUsername')]",
                    "linuxConfiguration": {
                        "disablePasswordAuthentication": true,
                        "ssh": {
                            "publicKeys": [
                                {
                                    "path": "[concat('/home/', parameters('adminUsername'), '/.ssh/authorized_keys')]",
                                    "keyData": "[parameters('sshPublicKey')]"
                                }
                            ]
                        }
                    }
                },
                "networkProfile": {
                    "networkInterfaces": [
                        {
                            "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('appServerNic'))]"
                        }
                    ]
                },
                "storageProfile": {
                    "imageReference": {
                        "publisher": "Canonical",
                        "offer": "0001-com-ubuntu-server-jammy",
                        "sku": "22_04-lts-gen2",
                        "version": "latest"
                    },
                    "osDisk": {
                        "createOption": "FromImage"
                    }
                }
            }
        },
        {
            "type": "Microsoft.Compute/virtualMachines",
            "apiVersion": "2020-06-01",
            "name": "BastionHost",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[resourceId('Microsoft.Network/networkInterfaces', variables('bastionHostNic'))]"
            ],
            "properties": {
                "hardwareProfile": {
                    "vmSize": "[parameters('vmSize')]"
                },
                "osProfile": {
                    "computerName": "BastionHost",
                    "adminUsername": "[parameters('adminUsername')]",
                    "linuxConfiguration": {
                        "disablePasswordAuthentication": true,
                        "ssh": {
                            "publicKeys": [
                                {
                                    "path": "[concat('/home/', parameters('adminUsername'), '/.ssh/authorized_keys')]",
                                    "keyData": "[parameters('sshPublicKey')]"
                                }
                            ]
                        }
                    }
                },
                "networkProfile": {
                    "networkInterfaces": [
                        {
                            "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('bastionHostNic'))]"
                        }
                    ]
                },
                "storageProfile": {
                    "imageReference": {
                        "publisher": "Canonical",
                        "offer": "0001-com-ubuntu-server-jammy",
                        "sku": "22_04-lts-gen2",
                        "version": "latest"
                    },
                    "osDisk": {
                        "createOption": "FromImage"
                    }
                }
            }
        }
    ]
}
```

[Click here to access the full ARM Template](#arm-template-for-infrastructure)

---

### **Deployment Script**
```bash
#!/bin/bash
set -e

resourceGroup="dotnetAppRG"
location="northeurope" 
templateFile="./IaaS-armTemplate.json"
adminUsername="azureuser"
vmSize="Standard_B1ls"

...

az deployment group create \
  --name "DotNetAppDeployment-$(date +%Y%m%d-%H%M%S)" \
  --resource-group "$resourceGroup" \
  --template-file "$templateFile" \
  --parameters \
    adminUsername="$adminUsername" \
    sshPublicKey="$sshPublicKey" \
    vmSize="$vmSize" \
    location="$location"
    
echo "Deployment completed successfully!"
```

[Click here to access the full Deployment Script](#deployment-script)

[Back to top](#table-of-contents)

---
