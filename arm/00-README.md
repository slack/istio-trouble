# AKS + Custom VNET + SLB

```console
az feature register --namespace "Microsoft.ContainerService" --name "VMSSPreview"
az feature register --namespace "Microsoft.ContainerService" --name "AKSAzureStandardLoadBalancer"
```

```console
export AKS_SUB=57ac26cf-a9f0-4908-b300-9a4e9a0fb205
export SPN_CLIENT_ID=

export AKS_RG=clusters
export AKS_NAME=istio2

export AKS_VNET_RG=clusters-vnets
export AKS_VNET_NAME=AKS-VNet-${AKS_NAME}
export AKS_VNET_RANGE=172.15.0.0/16
export AKS_SUBNET1_NAME=VNET-Local
export AKS_SUBNET1_RANGE=172.15.0.0/22
export AKS_SUBNET2_NAME=AKS-Nodes1
export AKS_SUBNET2_RANGE=172.15.4.0/22
export AKS_SUBNET3_NAME=AKS-Nodes2
export AKS_SUBNET3_RANGE=172.15.8.0/22

export AKS_SVC_RANGE=172.15.12.0/22
export AKS_BRIDGE_IP=172.16.0.1/24
export AKS_DNS_IP=172.15.12.2
```

## Create VNET Resources

```console
az group deployment create -n 01-aks-vnet -g ${AKS_VNET_RG} --template-file 01-aks-vnet.json \
    --parameters \
    vnetName=${AKS_VNET_NAME} \
    vnetAddressPrefix=${AKS_VNET_RANGE} \
    subnet1Name=${AKS_SUBNET1_NAME} \
    subnet1Prefix=${AKS_SUBNET1_RANGE} \
    subnet2Name=${AKS_SUBNET2_NAME} \
    subnet2Prefix=${AKS_SUBNET2_RANGE} \
    subnet3Name=${AKS_SUBNET3_NAME} \
    subnet3Prefix=${AKS_SUBNET3_RANGE}
```

## Grant AKS SP Access to VNET RG

```console
az role assignment create --role="Network Contributor" --scope=/subscriptions/${AKS_SUB}/resourceGroups/${AKS_VNET_RG} --assignee ${SPN_CLIENT_ID}
```

## Create AKS Cluster

```console
az group deployment create -n 02-aks -g ${AKS_RG} --template-file 02-aks.json \
    --parameters @03-parameters.json \
    --parameters \
    resourceName=${AKS_NAME} \
    dnsPrefix=${AKS_NAME} \
    networkPlugin=azure \
    serviceCidr=${AKS_SVC_RANGE} \
    dnsServiceIP=${AKS_DNS_IP} \
    dockerBridgeCidr=${AKS_BRIDGE_IP} \
    vnetSubnetIDNodes1=/subscriptions/${AKS_SUB}/resourceGroups/${AKS_VNET_RG}/providers/Microsoft.Network/virtualNetworks/${AKS_VNET_NAME}/subnets/${AKS_SUBNET2_NAME} \
    vnetSubnetIDNodes2=/subscriptions/${AKS_SUB}/resourceGroups/${AKS_VNET_RG}/providers/Microsoft.Network/virtualNetworks/${AKS_VNET_NAME}/subnets/${AKS_SUBNET3_NAME}
```

## GET AKS Cluster

```console
az resource show --api-version 2018-03-31 --id /subscriptions/${AKS_SUB}/resourceGroups/${AKS_RG}/providers/Microsoft.ContainerService/managedClusters/${AKS_NAME}
```
