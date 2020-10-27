# Objectives

* Learn how to create secure ML platform using Azure Machine Learning
    1. Provision Secure workspace
    1. Establish the access to private link enabled workspace
    1. Enable ML Studio UX features
    1. Provision Secure training env (Compute Cluster, Compute Instance)
    1. Provision Secure scoring env (AKS)
    1. Manage Users and Roles
    1. Data Security (TBD)
    1. Test machine learning job on secure AML platform
    1. Additional Configurations (Limit Outbound, Firewall, DNS)
* Roadmap discussion

## 0. Requirements

* Azure Subscription
* Submit a support request for private endpoint allowance. https://docs.microsoft.com/en-us/azure/machine-learning/how-to-manage-quotas#private-endpoint-and-private-dns-quota-increases
* Latest version of Azure CLI: sudo apt-get update && sudo apt-get install --only-upgrade -y azurecli
* Latest version of Azure CLI ML: az extension update -n azure-cli-ml

## 1. Provision Secure workspace

Output **TBU**

* Resource Group
* VNet and Subnets
* Private Link enabled workspace
* Storage with private endpoint
* ACR with private endpoint
* KV with private endpoint

### Resource Group

#### Output

* Resource Group to put workspace and related resources

#### Code

az group create -n ws1103 -l eastus 

### VNet and Subnets

#### Output

I plan to put workspace and training compute resources such as compute cluster, compute instance in training subnet. I also plan to put AKS cluster in the scoring subnet because AKS consumes many IPs.

* VNet: hub / 10.150.0.0/16
* Subnet: training / 10.150.0.0/24
* Subnet: scoring / 10.150.1.0/24
* Network Security Group for both subnets

#### Code

```sh
az network VNet create -g ws1103 -n hub --address-prefix 10.150.0.0/16 --subnet-name training --subnet-prefix 10.150.0.0/24
az network VNet subnet update -g ws1103 --VNet-name hub -n training --service-endpoints Microsoft.Storage Microsoft.KeyVault Microsoft.ContainerRegistry
az network VNet subnet create -g ws1103 --VNet-name hub -n scoring --address-prefixes 10.150.1.0/24
az network VNet subnet update -g ws1103 --VNet-name hub -n scoring --service-endpoints Microsoft.Storage Microsoft.KeyVault Microsoft.ContainerRegistry
az network nsg create -n ws1103nsg -g ws1103 
az network vnet subnet update -g ws1103 --vnet-name hub -n training --network-security-group ws1103nsg
az network vnet subnet update -g ws1103 --vnet-name hub -n scoring --network-security-group ws1103nsg

```

### KeyVault for encryption

#### Output

* KeyVault
* Key for Workspace Encryption

#### Code

```sh
az keyvault create -l eastus -n ws1103kv -g ws1103
az keyvault key create -n ws1103key --vault-name ws1103kv
```

> **BEFORE YOU CONTINUE** I recommend taking below information for workspace encryption parameters.

* Resource ID: /subscriptions/a4393d89-7e7f-4b0b-826e-72fc42c33d1f/resourceGroups/ws1103/providers/Microsoft.KeyVault/vaults/ws1103kv
* KID: https://ws1103kv.vault.azure.net/keys/ws1103key/5cb33f32dbbb468eb71c6ee0de1b5e84

### Provision Private Link and Customer Managed Key Enabled Workspace

#### Output

In your resource group,

* workspace and associated resources (Storage, KeyVault, ACR, AppInsights)
* Private Endpoint and NIC for PE
* Two Private DNS Zones (One for Workspace, One for Integrated Notebook)

In newly created resource group,

* CosmosDB and Storage to store your metadata
* Azure Search
* VNet (managed by Microsoft)

I recommend using this template and you can also create private link/CMK workspace from portal.azure.com.

* Template: https://github.com/Azure/azure-quickstart-templates/tree/master/201-machine-learning-advanced
* How to use doc: https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-workspace-template?tabs=azcli

You can find my examples in here. Below are key parameters.

* StorageAccount/KeyVault/ACR/VNet/Subnet Option: new or existing to use your existing resources
* StorageAccount/KeyVault/ACR BehindVNet: true or false
* containerRegistrySKU: Premium is required for behind VNet
* confidential data: hbi flag https://docs.microsoft.com/en-us/azure/machine-learning/concept-enterprise-security#encryption-at-rest
* cmk_keyvault: keyvault's resource id for cmk encryption
* resource_cmk_uri: key's resource id for cmk encryption
* privateEndpointType: AutoApproval/ManualApproval for private endpoint creation

Please note that you can create private endpoint for existing workspace from portal.azure.com or ARM template.

#### Code

> **WARNING** Don't forget to move the the folder that has arm template and parameters file.

```sh
az deployment group create -n plcmkworkspace1103 -g ws1103 -f azuredeploy.json -p azuredeploy.parameters.json
```

## 2. Establish the access to private link enabled workspace

Your private link enabled workspace can be accessed only through private endpoint created for your workspace over private IP of your VNet. In order to access your workspace, I explain two ways.

> **WARNING** If you see below error, most likely you are accessing private link enabled workspace using public IP. This is because your DNS configuration is not correct or you are accessing workspace from outside VNet.

> **REQUEST_SEND_ERROR**: Your request for data wasn’t sent. Here are some things to try: Check your network and internet connection, make sure a proxy server is not blocking your connection, follow our guidelines if you’re using a private link, and check if you have AdBlock turned on.

1. Establish Point to Site VPN connection to your workspace hub VNet and update hostfile
1. Create VM in your workspace hub VNet as a jumpbox VM

### Establish point to site VPN connection

Output

* Virtual network gateway
* Gateway subnet for Virtual network gateway
* Public IP addresses for Virtual network gateway

Documentation
https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal

Host file update **TBU**
https://docs.microsoft.com/en-us/azure/machine-learning/how-to-custom-dns?tabs=azure-cli

### Create VM in your workspace hub VNet

Output

* DSVM and VM related resources

Documentation

https://docs.microsoft.com/en-us/azure/machine-learning/data-science-virtual-machine/provision-vm

https://docs.microsoft.com/en-us/azure/bastion/create-host-cli

## 3. Enable ML Studio UX features

Services on ml.azure.com require the access right to your storage to access data. AML uses workspace managed identity for that. You need to access ml.azure.com, go to datastore and update default storage setting of "Click the Use workspace managed identity for data preview and profiling in Azure Machine Learning studio". Details steps are documented in below. https://docs.microsoft.com/en-us/azure/machine-learning/how-to-enable-studio-virtual-network

Please confirm data profiling works well with creating your dataset with your local CSV file. E.g.) https://www.kaggle.com/pavansubhasht/ibm-hr-analytics-attrition-dataset

> **WARNING** If you encounter the error when you upload dataset to storage, you do not have the access to storage. If you encounter the error to do data profiling, managed identity configuration of storage has issue.

## 4. Provision Secure training env (Compute Cluster, Compute Instance)

### Current Service Availability as of Oct 2020

| Service   |      Status      |
|----------|-------------|
| Compute Cluster behind VNet |  Fully supported |
| Compute Cluster with Private IP |    Private Preview but open to everyone in public regions, ARM template only   |
| Compute Instance behind VNet | Fully supported |
| Compute Instance with Private IP | Not Available |

### Output

* Compute Cluster behind VNet as CPU Cluster
* Compute Cluster with Private IP as GPU Cluster
* Compute Instance behind VNet

### Compute Cluster behind VNet

Compute cluster and Compute instance require inbound access from service tag BatchNodeManagement and AzureMachineLearning. At first let's allow those two service tags in inbound access in NSG.

```dotnetcli
az network nsg rule create -n batch --nsg-name ws1103nsg -g ws1103 --direction Inbound --priority 400 --source-address-prefixes BatchNodeManagement --source-port-ranges '*' --destination-port-ranges 29876-29877 --protocol Tcp --access Allow
az network nsg rule create -n aml --nsg-name ws1103nsg -g ws1103 --direction Inbound --priority 410 --source-address-prefixes AzureMachineLearning --source-port-ranges '*' --destination-port-ranges 44224 --protocol Tcp --access Allow
```

Then, create compute cluster behind VNet.

```dotnetcli
az ml computetarget create amlcompute -n cpu-cluster --min-nodes 1 --max-nodes 8 -s STANDARD_D3_V2 --vnet-name hub --vnet-resourcegroup-name ws1103 --subnet-name training --workspace-name ws1103 -g ws1103
```

Documentation
https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-attach-compute-cluster?tabs=python

> **WARNING** From UX, subnet is limited with workspace default VNet and subnet. Please use ARM template if you want to create it in different subnet under workspace default VNet. Please note that it should belong to the workspace VNet.

### Compute Cluster with Private IP

Currently creation using ARM template is only supported. Please use this ARM template and Parameter File.

At first you need to disable private endpoint network policy. Details in here. https://docs.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy

```dotnetcli
az network vnet subnet update -g ws1103 --vnet-name hub -n training --disable-private-endpoint-network-policies true --disable-private-link-service-network-policies true
```

Then, create private IP enabled compute cluster.

```dotnetcli
az deployment group create -n privateipcomputecluster -g ws1103 -f deployplcompute.json -p deployplcompute.parameters.json
```

### Compute Instance behind VNet

```dotnetcli
az ml computetarget create computeinstance -n ws1103ci1 -s "STANDARD_D3_V2" --vnet-name hub --vnet-resourcegroup-name ws1103 --subnet-name training --workspace-name ws1103 -g ws1103
```

Documentation
https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-manage-compute-instance?tabs=python

> **WARNING** From UX, subnet is limited with workspace default VNet and subnet. Please use ARM template if you want to create it in different subnet under workspace default VNet. Please note that it should belong to the workspace VNet.

## 5. Provision Secure scoring env (AKS)

| Service   |      Status      |
|----------|-------------|
| AKS behind VNet |  Fully supported. You can create it on ml.studio.com, python SDK, CLI, ARM. |
| Private AKS Cluster |    Attach is supported. Please create private AKS cluster at first and attach to AML workspace.   |
| Load Balancer | Both public load balancer (default) and private load balancer are supported. |

### AKS behind VNet with internal load balancer

```dotnetcli
az ml computetarget create aks -n ws1103aks1 --load-balancer-type InternalLoadBalancer --load-balancer-subnet scoring -l eastus -g ws1103 --vnet-name hub --vnet-resourcegroup-name ws1103 --subnet-name scoring --workspace-name ws1103 --cluster-purpose DevTest --service-cidr 10.0.0.0/16 --dns-service-ip 10.0.0.10 --docker-bridge-cidr 172.17.0.1/16 --vm-size 
```

### Private AKS Cluster with internal load balancer

```dotnetcli

```

## 6. Manage Users and Roles

Output
* Custom roles for IT Admin, Data Scientist, Super Data Scientist
* New account and grand Data Scientist role



## 7. Test machine learning job on secure AML platform

Now secure AML is ready to use.

## 8. Additional Configurations

### DNS

### Limit Outbound

### Firewall