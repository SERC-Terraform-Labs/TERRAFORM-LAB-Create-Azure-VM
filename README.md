# Terraform Lab - Create Azure VM

<img alt="points bar" align="right" height="36" src="../../blob/status/.github/activity-icons/points-bar.svg" />

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/.../...)

(You might need to wait a minute and refresh the page before the Gitpod link works.)

> :warning: This lab creates resources in Azure. These resources are covered under the student free tier. If your free tier has expired or you have used up all of your free credit, there will be a small charge to your account.

In this lab you will learn how to create a virtual machine in Azure using Terraform.

## Azure Virtual Machines

Using Terraform to create a virtual machine in Azure is a little bit more complicated than creating a virtual machine in AWS. To create a VM in Azure, the following also have to be created:

- resource group
- virtual network
- subnet
- network interface
- finally, the virtual machine

There are two resource types for creating virtual machines: one for Linux and one for Windows machines. We will by using the [Linux virtual machine](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_virtual_machine) resource.

## Azure CLI Login

Terraform must authenticate to Azure to create infrastructure. In your terminal, use the Azure CLI tool to setup your account permissions locally.

```bash
az login
```

Your browser window will open or you will be prompted to open a web page and enter a code. You will then be prompted to enter your Azure login credentials. After successful authentication, your terminal will display your subscription information. You do not need to save this output as it is saved in your system for Terraform to use.

## Write Configuration

### Resource Provider

Create a new file called `main.tf` and paste the following configuration:

```bash
# Configure the Microsoft Azure Provider
terraform {
    required_providers {
        azurerm = {
        source  = "hashicorp/azurerm"
        version = "~>2.0"
        }
    }
}
provider "azurerm" {
    features {}
}
```

### Resource Group

Create a new file called `variables.tf`. Copy and paste the variable declaration below:

```bash
variable "resource_group_name" {
    default = "terraform-create-vm-TEST"
}
```

This declaration includes a default value for the variable, so the resource_group_name variable will not be a required input.

In `main.tf` add the following block:

```bash
# Create resource group
resource "azurerm_resource_group" "terraformrg" {
    name     = var.resource_group_name
    location = "eastus"

    tags = {
        environment = "Terraform Demo"
    }
}
```

### Virtual Network

We need to create a virtual network for our VM. Add the following block to `main.tf`. Note the use of variables for some of the settings.

```bash
# Create virtual network
resource "azurerm_virtual_network" "terraformnetwork" {
    name                = "demoVnet"
    address_space       = ["10.0.0.0/16"]
    location            = "eastus"
    resource_group_name = azurerm_resource_group.terraformrg.name

    tags = {
        environment = "Terraform Demo"
    }
}
```

### Subnet

Create a subnet. Add the following block to `main.tf`:

```bash
# Create subnet
resource "azurerm_subnet" "terraformsubnet" {
    name                 = "demoSubnet"
    resource_group_name  = azurerm_resource_group.terraformrg.name
    virtual_network_name = azurerm_virtual_network.terraformnetwork.name
    address_prefixes       = ["10.0.1.0/24"]
}
```

### Network Interface

The VM needs a network interface. Add the following block to `main.tf`:

```bash
# Create network interface
resource "azurerm_network_interface" "terraformnic" {
    name                = "demoNIC"
    location            = "eastus"
    resource_group_name = azurerm_resource_group.terraformrg.name

    ip_configuration {
        name                          = "internal"
        subnet_id                     = azurerm_subnet.terraformsubnet.id
        private_ip_address_allocation = "Dynamic"
    }

    tags = {
        environment = "Terraform Demo"
    }
}
```

### Virtual Machine

We can now create our virtual machine. Add the following block to `main.tf`:

```bash
# Create virtual machine
resource "azurerm_linux_virtual_machine" "terraformvm" {
    name                  = "terraformVM"
    location              = "eastus"
    resource_group_name   = azurerm_resource_group.terraformrg.name
    network_interface_ids = [azurerm_network_interface.terraformnic.id]
    size                  = "Standard_B1s"

    os_disk {
        caching              = "ReadWrite"
        storage_account_type = "Standard_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "18.04-LTS"
        version   = "latest"
    }

    computer_name  = "demovm"
    admin_username = "azureuser"
    disable_password_authentication = false
    admin_password = "NotGoodPassword1234"

    tags = {
        environment = "Terraform Demo"
    }
}
```

The `azurerm_linux_virtual_machine` has a number of sub-blocks and settings:

- `name` - (Required) The name of the Linux Virtual Machine.
- `size` - (Required) The SKU which should be used for this Virtual Machine.
- `os_disk` - (Required) A os_disk block. This requires at least `caching` and `storage_account_type` settings. Full details are in the [provider documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_virtual_machine#caching).
- `admin_username` - (Required) The username of the local administrator used for the Virtual Machine.
- `source_image_reference` - One of either `source_image_id` or `source_image_reference` must be set. `source_image_reference` provides a more human readable way to define the image to use for the VM.

## Initialise and Apply

At this point we can deploy the virtual machine. Initialise the directory with the following command:

```bash
terraform init
```

Run the terraform apply command to apply your configuration.

```bash
terraform apply
```

Type `yes` at the confirmation prompt to proceed.

Wait for the configuration to be deployed, then open the [Azure portal](https://portal.azure.com) and navigate to our VM ( Resource groups > terraform-create-vm-TEST > terraformVM ).

## Connect to VM

At this point our VM exists, but we can't connect to it from outside of the virtual private network we created. We need to add a public IP address to the VM.

### Create Public IP Address

We need to create a public IP address. Add the following block to `main.tf`:

```bash
# Create public IPs
resource "azurerm_public_ip" "terraformpublicip" {
    name                         = "publicIP"
    location                     = "eastus"
    resource_group_name          = azurerm_resource_group.terraformrg.name
    allocation_method            = "Dynamic"

    tags = {
        environment = "Terraform Demo"
    }
}
```

### Add Public IP to NIC

Add the public address to the network interface IP configuration. Modify the `azurerm_network_interface` block in `main.tf`. Add the `public_ip_address_id` setting to the `ip_configuration` sub-block:

```diff
# Create network interface
resource "azurerm_network_interface" "terraformnic" {
    name                = "demoNIC"
    location            = "eastus"
    resource_group_name = azurerm_resource_group.terraformrg.name

    ip_configuration {
+        name                          = "internalANDexternal"
        subnet_id                     = azurerm_subnet.terraformsubnet.id
        private_ip_address_allocation = "Dynamic"
+        public_ip_address_id          = azurerm_public_ip.terraformpublicip.id
    }

    tags = {
        environment = "Terraform Demo"
    }
}
```

### Output Public IP Address

We will need to know the public IP address if we are to connect to the VM. Create an 'outputs.tf' file and add the following block:

```bash
# Output public IP address
output "public_ip" {
    value = azurerm_linux_virtual_machine.terraformvm.public_ip_address
}
```

### Try Connecting

Apply your changes to the configuration. You will need to destroy the infrastructure and then re-apply:

```bash
terraform destroy
terraform apply
```

When the deployment is finished the public IP address should be output. Use ssh to connect to the VM:

```
ssh azureuser@<output-ip-address>
```

When prompted, enter the password `NotGoodPassword1234`. You should now be logged into the Azure VM. This is possible because port 22 (SSH) is open by default.

## Use Security Group to Set Firewall Rules

It is good practice to set explicitly the firewall rules for our VM.

### Create Network Security Group and Access Rule

We need to create a network security group and then add security rules to it. Create a new file called `network-security.tf` and the following block to it:

```bash
# Create Network Security Group and rule
resource "azurerm_network_security_group" "terraformnsg" {
    name                = "myNetworkSecurityGroup"
    location            = "eastus"
    resource_group_name = azurerm_resource_group.terraformrg.name

    security_rule {
        name                       = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
    }

    tags = {
        environment = "Terraform Demo"
    }
}
```

We can include one security rule within this block. Here we have included a security rule to allow SSH access.

### Add Security Group To NIC

Next we need to add the security group to the VMs network interface. Add the following block to `network-security.tf`:

```bash
# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
    network_interface_id      = azurerm_network_interface.terraformnic.id
    network_security_group_id = azurerm_network_security_group.terraformnsg.id
}
```

### Try Connecting

Apply your changes to the configuration. Re-run `terraform apply`:

```bash
terraform apply
```

When the deployment is finished the public IP address should be output again. Use ssh to connect to the VM:

```
ssh azureuser@<output-ip-address>
```

## Use SSH Keys

Using a password for SSH connection isn't the most secure way. It would be better if we use a key pair.

### Generate SSH Keys

Create a file called `key-gen.tf` and add the following block:

```bash
# Create (and display) an SSH key
resource "tls_private_key" "example_ssh" {
    algorithm = "RSA"
    rsa_bits = 4096
}
output "tls_private_key" { 
        value = tls_private_key.example_ssh.private_key_pem 
        sensitive = true
}

# Write SSH key to file
resource "local_file" "private_key" {
    content         = tls_private_key.example_ssh.private_key_pem
    filename        = "azurevm.pem"
    file_permission = "0600"
}
```

This will create an SSH key pair for us and output the private key to a `azurevm.pem` file. (If you are on Windows, follow the instructions on [Stack Exchange](https://superuser.com/a/1329702) for setting the file permissions correctly.)

## Add Key to VM

The public key needs to be added to the VM. Additionally, we need to disable password login. Modify the `azurerm_linux_virtual_machine` block in `main.tf`:

```diff
# Create virtual machine
resource "azurerm_linux_virtual_machine" "terraformvm" {
    name                  = "terraformVM"
    location              = "eastus"
    resource_group_name   = azurerm_resource_group.terraformrg.name
    network_interface_ids = [azurerm_network_interface.terraformnic.id]
    size                  = "Standard_B1s"

    os_disk {
        caching              = "ReadWrite"
        storage_account_type = "Standard_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "18.04-LTS"
        version   = "latest"
    }

    computer_name  = "demovm"
    admin_username = "azureuser"
-    disable_password_authentication = false
+    disable_password_authentication = true
-    admin_password = "NotGoodPassword1234"

+    admin_ssh_key {
+        username       = "azureuser"
+        public_key     = tls_private_key.example_ssh.public_key_openssh
+    }
+
    tags = {
        environment = "Terraform Demo"
    }
}
```

### Try Connecting

Apply your changes to the configuration. Re-run `terraform apply`:

```bash
terraform apply
```

When the deployment is finished the public IP address should be output again. Use ssh to connect to the VM, this time using the key file:

```
ssh azureuser@<output-ip-address> -i azurevm.pem
```

## Destroy

The last step is to de-provision the infrastructure we have created. Run terraform destroy in the terminal:

```bash
terraform destroy
```

Finally, check in the [Azure portal](https://portal.azure.com/#blade/HubsExtension/BrowseResourceGroups) that the resource group has been deleted.
