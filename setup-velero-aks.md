## Setup Velero for AKS Restore

This section describes how to install and use Velero to restore workload backed up from EKS cluster using Azure Blob Storage

Open PowerShell as an administrator.

Run the following command to log in to Azure interactively:

```azcli
az login
```
Use the subscription name to find the subscription ID:

```azcli
$AZURE_SUBSCRIPTION_ID="<Your_Azure_SUBSCRIPTION_ID>"
```
Set the Azure subscription"

```azcli 
az account set -s $AZURE_BACKUP_SUBSCRIPTION_ID 
```

#### Create Azure Storage Account & Blob Container

Here we will create a storage account and a blob container to store the backup created in aws bucket.

Create an storage account to store backups from aws bucket

```azcli
$RESOURCE_GROUP="myResourceGroup"

$AZURE_STORAGE_ACCOUNT_ID="velerostorageaccount"
az storage account create --name $AZURE_STORAGE_ACCOUNT_ID --resource-group $RESOURCE_GROUP --sku Standard_GRS --encryption-services blob --https-only true --kind BlobStorage --access-tier Hot
```

Create a blob container:
```azcli
$BLOB_CONTAINER="velero"
az storage container create -n $BLOB_CONTAINER --public-access off --account-name $AZURE_STORAGE_ACCOUNT_ID
```
#### Create a Service Principal to grant permission to velero

Get the subscription ID and tenant ID for your Azure account:

```azcli
$AZURE_SUBSCRIPTION_ID=(az account list --query '[?isDefault].id' -o tsv)
$AZURE_TENANT_ID=(az account list --query '[?isDefault].tenantId' -o tsv)
```

##### Create a service principal

You can create a service principal with the Contributor role or use a custom role:

Contributor role: The Contributor role grants subscription-wide access, so be sure protect this credential if you assign that role.
Custom role: If you need a more restrictive role, use a custom role.

To create a service principal with the Contributor role, use the following command. Substitute your own subscription ID and, optionally, your own service principal name. Microsoft Entra ID will generate a secret for you.

```azcli
$AZURE_CLIENT_SECRET=(az ad sp create-for-rbac --name "azvelerosp" --role "Contributor" --query 'password' -o tsv --scopes  /subscriptions/$AZURE_SUBSCRIPTION_ID)
```

Get the service principal name, and assign that name to the `AZURE_CLIENT_ID` variable:

```azcli
$AZURE_CLIENT_ID=(az ad sp list --display-name "azvelerosp" --query '[0].appId' -o tsv)
```

Create a file that contains the variables the Velero installation requires. The command looks similar to the following one:

```
AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
AZURE_TENANT_ID=$AZURE_TENANT_ID
AZURE_CLIENT_ID=$AZURE_CLIENT_ID
AZURE_CLIENT_SECRET=$AZURE_CLIENT_SECRET
AZURE_RESOURCE_GROUP=$RESOURCE_GROUP
AZURE_CLOUD_NAME=AzurePublicCloud" | Out-File -FilePath ./credentials-velero.txt
```

### Installing Velero server in AKS Cluster

Install Velero on the cluster, and start the deployment. This procedure creates a namespace called velero and adds a deployment named velero to the namespace.

```azcli
velero install --provider azure --plugins velero/velero-plugin-for-microsoft-azure:v1.5.0 --bucket $BLOB_CONTAINER --secret-file ./credentials-velero.txt --backup-location-config resourceGroup=$RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_ID,subscriptionId=$AZURE_SUBSCRIPTION_ID --uploader-type "restic" --use-volume-snapshots --use-node-agent
```
To check whether the Velero service is running properly First, switch context to the recovery context that we created earlier by using this command:


```powershell
$RECOVERY_CONTEXT="aks_restore_velero"
kubectl config use-context $RECOVERY_CONTEXT

kubectl -n velero get pods
kubectl logs deployment/velero -n velero
```

The get pods command should show that the Velero pods are running as shown below:

```bash
NAME                      READY   STATUS    RESTARTS   AGE
node-agent-28lx7          1/1     Running   0          2d
node-agent-d78wq          1/1     Running   0          2d
node-agent-htrrw          1/1     Running   0          2d
velero-6686c4848f-nr487   1/1     Running   0          2d
```
Check if the Backup Storage Location is in “Available” state: 

```bash
velero backup-location get

NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   azure      velero          Available   2024-02-02 12:22:04 +0530 IST   ReadWrite     true
```

### Next Step
[Transfer Backup Data from AWS S3 Bucket and Restore in Azure blob storage using AzCopy](copy-data-using-Azcopy.md)