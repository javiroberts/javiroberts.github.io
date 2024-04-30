---
title: "Deploying an AKS Cluster Using Terraform"
date: 2023-03-13T14:10:29-03:00
tags:
  - development
  - devops
  - infrastructure
categories:
  - tech
draft: false
summary: A detailed breakdown on how to deploy an AKS cluster using Terraform.
---

In this post, I will be exploring how to use Terraform to deploy an AKS cluster in just a few easy steps. By the end of this post, you will have a fully functional AKS cluster up and running, ready to deploy your applications and services.

## Requirements

First, a few things that we'll need prior to getting started:
- Full access to your Azure account (this includes role and resource creation permissions)
- A working installation of Azure CLI
- A working installation of Terraform

## Hands On

So, having all of those requirements up and running, lets get started with the process. The outline will be the following:
1. Create a manually managed storage account for Terraform state backend
2. Create a resource group to assign all created resources to
3. Write all the networking resource definitions
4. Write all the Kubernetes resource definitions

### Create a storage backend

To get started we will create an storage account to save the Terraform state to. This is a more scalable approach than just saving that backend file to your PC and somehow trying to manage it afterwards. Of course this step is predecessor to managing terraform resources, so for this, we will use Azure CLI.

The first step is to get our subscription ID, this can be done with the following command:
```shell
az account show

{
  "environmentName": "AzureCloud",
  "homeTenantId": <your_homeTenantId>,
  "id": <subscriptionId>,
  "isDefault": true,
  ...
  "tenantId": <your_tenantId>,
  "user": {
    "name": <your_name>,
    "type": "user"
  }
}
```
From the output, get the ID field of the account you would like to use for that cluster.

After that we will create a service principal for the account. This is a special type of credentials that let you authenticate on behalf of an app instead of a regular user. For that we execute the following command:
```shell
az ad sp create-for-rbac --role="Contributor" -n="<your_app_name>" --scopes="/subscriptions/<subscriptionId>"

{
  "appId": "<appId>",
  "displayName": "<your_app_name>",
  "password": "<password>",
  "tenant": "<tenant>"
  ...
}
```
From that command's output my recommendation is that you set an ENV file or a shell script to hold those values and keep them at hand.

So, for that let's create a new file called `env.sh` in the root folder of our project. Then, populate that file with the following content:
```shell
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_SUBSCRIPTION_ID="<subscriptionId>"
export ARM_TENANT_ID="<tenant>"
```
This file is not the safest way to store sensitive information, but, for this project it will suffice, just make sure to add it to your `.gitignore` to avoid committing any sensitive information to your repository.

In the next step we will create a script named `terraform-backend.sh` that will be in charge of creating the backend:
```shell
RESOURCE_GROUP_NAME=tfstate
STORAGE_ACCOUNT_NAME=tfstate$RANDOM
CONTAINER_NAME=tfstate

# Create resource group
az group create --name $RESOURCE_GROUP_NAME --location eastus

# Create storage account
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

# Create blob container
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME
```
In this case this creates it in the `eastus` region, but you can choose your own. Just make sure that it's the same region you will use for the rest of the project. After executing this we will be ready for the final step of this section, the initial Terraform definition. Write down your command outputs, they will come in handy on the next step.

Create a file named `main.tf` in your root dir. Here we will add some initial definitions:
```terraform
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "tfstate18325"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0.2"
    }
  }

  required_version = ">= 1.1.0"
}

provider "azurerm" {
  features {}
}
```
In the main `terraform { ... }` block we are defining the general config for the Terraform, there we are adding our backend configuration.

If you used the script provided in the previous step, the only thing you will have to replace is the `storage_account_name` with your own randomly generated name. Otherwise, input your own values.

The `provider "azurerm" { ... }` block, tells terraform to install that dependency, this way we can directly manage azure infrastructure.

> `azurerm` provider takes it's config values from the variables prefixed with `ARM_`. This variables are set in the `env.sh` script that we previously created.

Once everything is set up you can run `terraform init` command to initialize backend storage and install the plugins.

> Two commands that will be very useful to keep your project consistent and readable, are `terraform fmt` to format the code, and `terraform validate` to lint and validate the files.
> My recommendation is that you run them periodically. They'll make your life easier!

### Create a resource group

Let's start by adding the following variables to our `env.sh` file:
```shell
{...}

export TF_VAR_LOCATION="eastus"
export TF_VAR_RESOURCE_GROUP_NAME="your-resource-group"
```
_Note that you can customize with your own values._

After that we will create a file named `variables.tf` with the following content:
```terraform
variable "LOCATION" {
  type        = string
  description = "Azure main working location. Can be set through ENV or a vars file."
}

variable "RESOURCE_GROUP_NAME" {
  type        = string
  description = "Azure resource group name. Can be set through ENV or a vars file."
}
```

> You might have noticed that the env variables previously defined are prefixed with `TF_VAR_`. This is the way terraform has to auto populate it's internal variables with shell environment variables.
> Another alternative is using a `.tfvars` file.

With all of these set we can proceed to define our resource group in our `main.tf`, just append the following:
```terraform
{...}

resource "azurerm_resource_group" "yourname" {
  name     = var.RESOURCE_GROUP_NAME
  location = var.LOCATION
}
```

If you wish you can run `terraform plan` to see how you are doing so far, just remember to source your `env.sh` first!

### Create the networks

_This section is completely optional. If you don't want to do it, you can skip to the next one though it is a good practice to define as much of the infrastructure as you can through terraform files._

First let's add some variables to our `env.sh`:
```shell
{...}

export TF_VAR_VIRTUAL_NETWORK_NAME="your-virtual-network"
```

And update our `variables.tf` accordingly:
```
{...}

variable "VIRTUAL_NETWORK_NAME" {
  type        = string
  description = "Azure virtual network name. Can be set through ENV or a vars file."
}
```

After that let's define some resources updating our `main.tf` file:
```terraform
{...}

resource "azurerm_virtual_network" "yourname" {
  name                = var.VIRTUAL_NETWORK_NAME
  location            = var.LOCATION
  resource_group_name = var.RESOURCE_GROUP_NAME
  address_space       = ["10.0.0.0/8"]

  depends_on = [
    azurerm_resource_group.vrj,
  ]
}

resource "azurerm_subnet" "yourname-bastion-subnet" {
  name                 = "bastion-subnet"
  resource_group_name  = var.RESOURCE_GROUP_NAME
  virtual_network_name = var.VIRTUAL_NETWORK_NAME
  address_prefixes     = ["10.88.0.0/16"]

  depends_on = [
    azurerm_resource_group.vrj,
    azurerm_virtual_network.vrj,
  ]
}

resource "azurerm_subnet" "yourname-aks-subnet" {
  name                 = "aks-subnet"
  resource_group_name  = var.RESOURCE_GROUP_NAME
  virtual_network_name = var.VIRTUAL_NETWORK_NAME
  address_prefixes     = ["10.77.0.0/16"]

  depends_on = [
    azurerm_resource_group.vrj,
    azurerm_virtual_network.vrj,
  ]
}
```
Here, we create a virtual network with a `/8` CIDR and two subnets, one for bastion/admin purpose with a `10.88.x.x/16` address space and another one with a `10.77.x.x/16` CIDR for Kubernetes.

At this point we are able to run `terraform fmt`, `terraform validate` and `terraform plan` to get a hold on our progress.

### Create cluster resources

At this step we will create the cluster resources. For this we will follow the same mechanics as before. Add variables to our env, add variable definitions to terraform, and update our main terraform file.

Let's start by defining the environment variables in `env.sh`:
```shell
{...}

export TF_VAR_ARM_CLIENT_ID=${ARM_CLIENT_ID}
export TF_VAR_ARM_CLIENT_SECRET=${ARM_CLIENT_SECRET}
export TF_VAR_KUBERNETES_CLUSTER_NAME="your-aks"
export TF_VAR_KUBERNETES_DNS_PREFIX="your-k8s"
```
Did you notice the first two variables we added? They are taking their value from the previously defined `ARM` values, this is for more consistency between different variables.

Next, we will add those variables to our `variables.tf`, so that we make them available for Terraform.
```terraform
{...}

variable "ARM_CLIENT_ID" {
  type        = string
  description = "Azure Client ID. Should be set through ENV."
}

variable "ARM_CLIENT_SECRET" {
  type        = string
  description = "Azure Client Secret. Should be set through ENV."
}

variable "KUBERNETES_CLUSTER_NAME" {
  type        = string
  description = "K8s cluster name. Can be set through ENV or a vars file."
}

variable "KUBERNETES_DNS_PREFIX" {
  type        = string
  description = "K8s DNS prefix. Can be set through ENV or a vars file."
}
```

And to our `main.tf` we will append the following block:
```terraform
{...}

resource "azurerm_kubernetes_cluster" "yourname" {
  name                              = var.KUBERNETES_CLUSTER_NAME
  location                          = var.LOCATION
  resource_group_name               = var.RESOURCE_GROUP_NAME
  dns_prefix                        = var.KUBERNETES_DNS_PREFIX
  role_based_access_control_enabled = true

  default_node_pool {
    name            = "default"
    node_count      = 3
    vm_size         = "Standard_B2s"
    os_disk_size_gb = 10
    vnet_subnet_id = azurerm_subnet.yourname-aks-subnet.id # omit if not defined
  }

  linux_profile {
    admin_username = "<yourusername>"
    ssh_key {
      key_data = "ssh-rsa <yourkeydata>"
    }
  }

  service_principal {
    client_id     = var.ARM_CLIENT_ID
    client_secret = var.ARM_CLIENT_SECRET
  }

  tags = {
    environment = "testing"
    iaac        = true
  }

  depends_on = [
    azurerm_resource_group.vrj,
    azurerm_virtual_network.vrj,
    azurerm_subnet.vrj-aks-subnet,
  ]
}
```

Let's break down the aforementioned block.

- First, we set the general params for the cluster such as name, region, etc
- Then we define a `node_pool` and it's characteristics. This has 3 nodes with 10GB each for OS disk. And most importantly, ties your node pool to it's subnet if defined.
- Afterwards through `linux_profile` we provide username and credentials to be able to log in to the nodes if needed.
- At last we provide the same `service_principal` we've been using so far.

### Wrapping up

You can see the whole files in the following [gist](https://gist.github.com/javiroberts/9d45297cb05b64b8353144259a46a09b). If your's are OK you can run the following commands:
```shell
terraform fmt
terraform validate
terraform apply
```
If everything went smooth your cluster should be up and running and you can configure `.kube/config` with the following command:
```
az aks get-credentials --resource-group $TF_VAR_RESOURCE_GROUP_NAME --name $TF_VAR_KUBERNETES_CLUSTER_NAME
```

> Keep in mind that if your `depends_on` sections are not properly set your cluster will fail it's creation.
> Always remember to source your `env.sh`!

## Closing statement

So far we've created a cluster and configured everything with terraform. In following posts we will install ArgoCD and create some CI/CD pipelines implementing GitOps methodology.

Thanks for reading. If you liked it or want to contact me, give me a follow in any of my social media or shoot me an email.
