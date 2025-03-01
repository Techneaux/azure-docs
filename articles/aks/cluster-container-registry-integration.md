---
title: Integrate Azure Container Registry with Azure Kubernetes Service
description: Learn how to integrate Azure Kubernetes Service (AKS) with Azure Container Registry (ACR)
services: container-service
ms.topic: article
ms.date: 11/16/2022
ms.tool: azure-cli, azure-powershell
ms.devlang: azurecli
---

# Authenticate with Azure Container Registry from Azure Kubernetes Service

You need to establish an authentication mechanism when using [Azure Container Registry (ACR)][acr-intro] with Azure Kubernetes Service (AKS). This operation is implemented as part of the Azure CLI, Azure PowerShell, and Azure portal experiences by granting the required permissions to your ACR. This article provides examples for configuring authentication between these Azure services.

You can set up the AKS to ACR integration using the Azure CLI or Azure PowerShell. The AKS to ACR integration assigns the [**AcrPull** role][acr-pull] to the [Azure Active Directory (Azure AD) **managed identity**][aad-identity] associated with your AKS cluster.

> [!NOTE]
> This article covers automatic authentication between AKS and ACR. If you need to pull an image from a private external registry, use an [image pull secret][image-pull-secret].

## Before you begin

* You need to have the [**Owner**][rbac-owner], [**Azure account administrator**][rbac-classic], or [**Azure co-administrator**][rbac-classic] role on your **Azure subscription**.
  * To avoid needing one of these roles, you can instead use an existing managed identity to authenticate ACR from AKS. For more information, see [Use an Azure managed identity to authenticate to an ACR](../container-registry/container-registry-authentication-managed-identity.md).
* If you're using Azure CLI, this article requires that you're running Azure CLI version 2.7.0 or later. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI][azure-cli-install].
* If you're using Azure PowerShell, this article requires that you're running Azure PowerShell version 5.9.0 or later. Run `Get-InstalledModule -Name Az` to find the version. If you need to install or upgrade, see [Install Azure PowerShell][azure-powershell-install].

## Create a new AKS cluster with ACR integration

You can set up AKS and ACR integration during the creation of your AKS cluster. To allow an AKS cluster to interact with ACR, an Azure AD managed identity is used.

### Create an ACR

If you don't already have an ACR, create one using the following command.

#### [Azure CLI](#tab/azure-cli)

```azurecli
# Set this variable to the name of your ACR. The name must be globally unique.

MYACR=myContainerRegistry

az acr create -n $MYACR -g myContainerRegistryResourceGroup --sku basic
```

#### [Azure PowerShell](#tab/azure-powershell)

```azurepowershell
# Set this variable to the name of your ACR. The name must be globally unique.

$MYACR = 'myContainerRegistry'

New-AzContainerRegistry -Name $MYACR -ResourceGroupName myContainerRegistryResourceGroup -Sku Basic
```

---

### Create a new AKS cluster and integrate with an existing ACR

If you already have an ACR, use the following command to create a new AKS cluster with ACR integration. This command allows you to authorize an existing ACR in your subscription and configures the appropriate **AcrPull** role for the managed identity. Supply valid values for your parameters below.

#### [Azure CLI](#tab/azure-cli)

```azurecli
# Set this variable to the name of your ACR. The name must be globally unique.

MYACR=myContainerRegistry

# Create an AKS cluster with ACR integration.

az aks create -n myAKSCluster -g myResourceGroup --generate-ssh-keys --attach-acr $MYACR
```

Alternatively, you can specify the ACR name using an ACR resource ID using the following format:

`/subscriptions/\<subscription-id\>/resourceGroups/\<resource-group-name\>/providers/Microsoft.ContainerRegistry/registries/\<name\>`

> [!NOTE]
> If you're using an ACR located in a different subscription from your AKS cluster, use the ACR *resource ID* when attaching or detaching from the cluster.
>
> ```azurecli
> az aks create -n myAKSCluster -g myResourceGroup --generate-ssh-keys --attach-acr /subscriptions/<subscription-id>/resourceGroups/myContainerRegistryResourceGroup/providers/Microsoft.ContainerRegistry/registries/myContainerRegistry
> ```

#### [Azure PowerShell](#tab/azure-powershell)

```azurepowershell
# Set this variable to the name of your ACR. The name must be globally unique.

$MYACR = 'myContainerRegistry'

# Create an AKS cluster with ACR integration.

New-AzAksCluster -Name myAKSCluster -ResourceGroupName myResourceGroup -GenerateSshKey -AcrNameToAttach $MYACR
```

---

This step may take several minutes to complete.

## Configure ACR integration for existing AKS clusters

### Attach an ACR to an AKS cluster

#### [Azure CLI](#tab/azure-cli)

Integrate an existing ACR with an existing AKS cluster using the [`--attach-acr` parameter][cli-param] and valid values for **acr-name** or **acr-resource-id**.

```azurecli
# Attach using acr-name
az aks update -n myAKSCluster -g myResourceGroup --attach-acr <acr-name>

# Attach using acr-resource-id
az aks update -n myAKSCluster -g myResourceGroup --attach-acr <acr-resource-id>
```

> [!NOTE]
> The `az aks update --attach-acr` command uses the permissions of the user running the command to create the ACR role assignment. This role is assigned to the [kubelet][kubelet] managed identity. For more information on AKS managed identities, see [Summary of managed identities][summary-msi].

#### [Azure PowerShell](#tab/azure-powershell)

Integrate an existing ACR with an existing AKS cluster using the [`-AcrNameToAttach` parameter][ps-attach] and valid values for **acr-name**.

```azurepowershell
Set-AzAksCluster -Name myAKSCluster -ResourceGroupName myResourceGroup -AcrNameToAttach <acr-name>
```

> [!NOTE]
> Running the `Set-AzAksCluster -AcrNameToAttach` cmdlet uses the permissions of the user running the command to create the role ACR assignment. This role is assigned to the [kubelet][kubelet] managed identity. For more information on AKS managed identities, see [Summary of managed identities][summary-msi].

---

### Detach an ACR from an AKS cluster

#### [Azure CLI](#tab/azure-cli)

Remove the integration between an ACR and an AKS cluster using the [`--detach-acr` parameter][cli-param] and valid values for **acr-name** or **acr-resource-id**.

```azurecli
# Detach using acr-name
az aks update -n myAKSCluster -g myResourceGroup --detach-acr <acr-name>

# Detach using acr-resource-id
az aks update -n myAKSCluster -g myResourceGroup --detach-acr <acr-resource-id>
```

#### [Azure PowerShell](#tab/azure-powershell)

Remove the integration between an ACR and an AKS cluster using the [`-AcrNameToDetach` parameter][ps-detach] and valid values for **acr-name**.

```azurepowershell
Set-AzAksCluster -Name myAKSCluster -ResourceGroupName myResourceGroup -AcrNameToDetach <acr-name>
```

---

## Working with ACR & AKS

### Import an image into your ACR

Run the following command to import an image from Docker Hub into your ACR.

#### [Azure CLI](#tab/azure-cli)

```azurecli
az acr import  -n <acr-name> --source docker.io/library/nginx:latest --image nginx:v1
```

#### [Azure PowerShell](#tab/azure-powershell)

```azurepowershell
Import-AzContainerRegistryImage -RegistryName <acr-name> -ResourceGroupName myResourceGroup -SourceRegistryUri docker.io -SourceImage library/nginx:latest
```

---

### Deploy the sample image from ACR to AKS

Ensure you have the proper AKS credentials.

#### [Azure CLI](#tab/azure-cli)

```azurecli
az aks get-credentials -g myResourceGroup -n myAKSCluster
```

#### [Azure PowerShell](#tab/azure-powershell)

```azurepowershell
Import-AzAksCredential -ResourceGroupName myResourceGroup -Name myAKSCluster
```

---

Create a file called **acr-nginx.yaml** using the sample YAML below. Replace **acr-name** with the name of your ACR.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx0-deployment
  labels:
    app: nginx0-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx0
  template:
    metadata:
      labels:
        app: nginx0
    spec:
      containers:
      - name: nginx
        image: <acr-name>.azurecr.io/nginx:v1
        ports:
        - containerPort: 80
```

After creating the file, run the following deployment in your AKS cluster.

```console
kubectl apply -f acr-nginx.yaml
```

You can monitor the deployment by running `kubectl get pods`.

```console
kubectl get pods
```

The output should show two running pods.

```output
NAME                                 READY   STATUS    RESTARTS   AGE
nginx0-deployment-669dfc4d4b-x74kr   1/1     Running   0          20s
nginx0-deployment-669dfc4d4b-xdpd6   1/1     Running   0          20s
```

### Troubleshooting

* Run the [`az aks check-acr`](/cli/azure/aks#az-aks-check-acr) command to validate that the registry is accessible from the AKS cluster.
* Learn more about [ACR monitoring](../container-registry/monitor-service.md).
* Learn more about [ACR health](../container-registry/container-registry-check-health.md).

<!-- LINKS - external -->

[image-pull-secret]: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
[summary-msi]: use-managed-identity.md#summary-of-managed-identities
[acr-pull]: ../role-based-access-control/built-in-roles.md#acrpull
[azure-cli-install]: /cli/azure/install-azure-cli
[azure-powershell-install]: /powershell/azure/install-az-ps
[acr-intro]: ../container-registry/container-registry-intro.md
[aad-identity]: ../active-directory/managed-identities-azure-resources/overview.md
[rbac-owner]: ../role-based-access-control/built-in-roles.md#owner
[rbac-classic]: ../role-based-access-control/rbac-and-directory-admin-roles.md#classic-subscription-administrator-roles
[kubelet]: https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
[ps-detach]: /powershell/module/az.aks/set-azakscluster#-acrnametodetach
[cli-param]: /cli/azure/aks#az-aks-update-optional-parameters
[ps-attach]: /powershell/module/az.aks/set-azakscluster#-acrnametoattach
