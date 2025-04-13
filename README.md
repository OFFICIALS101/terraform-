# terraform-
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "terraform-vnet-rg"
  location = "West Europe"
}

resource "azurerm_virtual_network" "vnet" {
  name                = "my-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "public" {
  name                 = "public-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}



resource "azurerm_subnet" "private" {
  name                 = "private-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_network_security_group" "nsg" {
  name                = "vm-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

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

  security_rule {
    name                       = "HTTP"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface" "nic_public" {
  name                = "nic-public"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.public.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.public_ip.id
  }
}

resource "azurerm_public_ip" "public_ip" {
  name                = "public-ip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Dynamic"
}

resource "azurerm_network_interface_security_group_association" "public_nic_nsg" {
  network_interface_id      = azurerm_network_interface.nic_public.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}

resource "azurerm_linux_virtual_machine" "vm_public" {
  name                = "nginx-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B1s"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic_public.id,
  ]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  custom_data = filebase64("${path.module}/user_data_nginx.sh")
}

resource "azurerm_network_interface" "nic_private" {
  name                = "nic-private"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.private.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_network_interface_security_group_association" "private_nic_nsg" {
  network_interface_id      = azurerm_network_interface.nic_private.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}

resource "azurerm_linux_virtual_machine" "vm_private" {
  name                = "postgres-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B1s"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic_private.id,
  ]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  custom_data = filebase64("${path.module}/user_data_postgres.sh")
}
.terraform/
*.tfstate
*.tfstate.backup
.terraform.lock.hcl

terraform init
terraform apply
# After testing
terraform destroy


#  🌐 Terraform Azure VNet Project

This project provisions an Azure Virtual Network (VNet) with public and private subnets, deploying two Ubuntu VMs:

Public VM with NGINX installed

Private VM with PostgreSQL installed

Resources are managed using Terraform and can be cleaned up easily with terraform destroy.

🛠️ Prerequisites

Azure CLI authenticated (az login)

Terraform installed

SSH key pair (~/.ssh/id_rsa.pub)

Azure Subscription

🗂️ Project Structure

terraform-azure-vnet/
├── main.tf
├── variables.tf
├── outputs.tf
├── user_data_nginx.sh
├── user_data_postgres.sh
├── .gitignore
└── README.md

⚙️ Step-by-Step Setup

1. Clone the repository

git clone https://github.com/your-username/terraform-azure-vnet.git
cd terraform-azure-vnet

2. Initialize Terraform

terraform init

3. Deploy Resources

terraform apply

Approve with yes.

4. Output – Public VM IP

After deployment, Terraform displays the public IP of the NGINX VM:

Apply complete!
Outputs:
public_vm_ip = "X.X.X.X"

5. Connect to the VM

ssh azureuser@X.X.X.X

6. Verify Installations

✅ NGINX (Public VM)

Open browser: http://<public-ip>You should see the default NGINX page.

✅ PostgreSQL (Private VM)

ssh into public VM → then use `ssh azureuser@<private_ip>` (if configured)
sudo systemctl status postgresql

7. Destroy Resources

To clean up:

terraform destroy


🧹 Resources Created

Resource Group

Virtual Network (VNet)

Public and Private Subnets

Network Security Group (NSG)

Public IP

2x Ubuntu Linux Virtual Machines

Custom Script Extensions for NGINX and PostgreSQL

📎 Notes

Public VM has HTTP & SSH open

Private VM is only accessible within the VNet

NGINX is available via public IP

PostgreSQL is running in isolation for backend usage
