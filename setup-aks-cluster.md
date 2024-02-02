## Deploy an Azure Kubernetes Service (AKS)

Azure Kubernetes Service (AKS) is a managed Kubernetes service that lets you quickly deploy and manage clusters. 

### Some Pre-Requisites:

An active Azure subscription. If you don't have one, create a free Azure account before you begin.
Azure CLI version 2.50.0 or later installed. to install or upgrade, see [Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).
`aks-preview Azure CLI extension of version 0.5.145 or later installed.

You can run az --version to verify above versions.

Apart from that, we need to have **kubectl** installed in our system too, for communicating with the Pods running on EKS. It is a managed service so everything will be managed by it except kubectl command because it is a client program, which will help us to connect with the pods.
[**Install and Set Up kubectl**
*The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters. You can use kubectl…*kubernetes.io](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### Starting the EKS Cluster

## Create a resource group

Open PowerShell as an administrator and run the following command:

```azurecli
$RESOURCE_GROUP="myResourceGroup"
$LOCATION="eastus"
$AKS_CLUSTER_NAME="myAKSCluster"
az group create --name $RESOURCE_GROUP --location $LOCATION
```

## Create an AKS cluster
To create an AKS cluster, use the az aks create command. The following example creates a cluster named myAKSCluster with one node and enables a system-assigned managed identity.

```azurecli
az aks create --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --enable-managed-identity --node-count 1 --generate-ssh-keys
```
this will take few minutes to complete the AKS cluster creation and returns JSON-formatted information about the cluster.

## Connect to the cluster

Configure `kubectl` to connect to your Kubernetes cluster using the `az aks get-credentials` command. This command downloads credentials and configures the Kubernetes CLI to use them.

```azurecli
$RECOVERY_CONTEXT="aks_restore_velero"
az aks get-credentials --resource-group myResourceGroup --name $AKS_CLUSTER_NAME --context $RECOVERY_CONTEXT
```
Verify the connection to your cluster using the `kubectl get`` command. This command returns a list of the cluster nodes.

```azurecli
kubectl get nodes
```
The following sample output shows the single node created in the previous steps. Make sure the node status is Ready.