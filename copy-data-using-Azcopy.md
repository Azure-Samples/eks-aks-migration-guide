
### Copy data from Amazon S3 to Azure Storage by using AzCopy

As we already have the velero client installed in the previous step lets copy the data from AWS bucket to Azure Blob Storage using AzCopy. AzCopy is a command-line utility that you can use to copy blobs or files to or from a storage account.

#### Download and install AzCopy

These files are compressed as a zip file (Windows and Mac) or a tar file (Linux). To download and decompress the tar file on Linux, see the documentation for your Linux distribution.

For detailed information on AzCopy releases, see the [AzCopy release page](https://github.com/Azure/azure-storage-azcopy/releases).

First, download the AzCopy executable file to any directory on your computer. AzCopy is just an executable file, so there's nothing to install.

[Windows 64-bit (zip)](https://aka.ms/downloadazcopy-v10-windows)
[Windows 32-bit (zip)](https://aka.ms/downloadazcopy-v10-windows-32bit)
[Linux x86-64 (tar)](https://aka.ms/downloadazcopy-v10-linux)
[Linux ARM64 (tar)](https://aka.ms/downloadazcopy-v10-linux-arm64)
[macOS (zip)](https://aka.ms/downloadazcopy-v10-mac)
[macOS ARM64 Preview (zip)](https://aka.ms/downloadazcopy-v10-mac-arm64)

##### Authorize with Azure Storage
Run the below command to Authorize with Azure Storage

```azcli
AzCopy login
```

#####  Authorize with Azure Storage

Gather your AWS access key and secret access key, and then set these environment variables:

| Operating system | Command  |
|--------|-----------|
| **Windows** | `set AWS_ACCESS_KEY_ID=<access-key>`<br>`set AWS_SECRET_ACCESS_KEY=<secret-access-key>` |
| **Linux** | `export AWS_ACCESS_KEY_ID=<access-key>`<br>`export AWS_SECRET_ACCESS_KEY=<secret-access-key>`|
| **macOS** | `export AWS_ACCESS_KEY_ID=<access-key>`<br>`export AWS_SECRET_ACCESS_KEY=<secret-access-key>`|

These credentials are used to generate pre-signed URLs that are used to copy objects.

**Copy a bucket**

```bash
azcopy copy 'https://s3.amazonaws.com/$BLOB_CONTAINER' 'https://$AZURE_STORAGE_ACCOUNT_ID.blob.core.windows.net/$BLOB_CONTAINER' --recursive=true
```
`$Bucket` is the name of aws s3 bucket that we have created to storage velero backup in previous step.
`$AZURE_STORAGE_ACCOUNT_ID` and `$BLOB_CONTAINER` is the name of Azure blob storage account and container name that we have created earlier in this article.

For more details, See [Copy data from Amazon S3 to Azure Storage by using AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-s3).

To verify the list of blob copied in the Azure storage container from aws s3 bucket run the below command:

```azcli
az storage blob list -c $BLOB_CONTAINER --account-name AZURE_STORAGE_ACCOUNT_ID --output table
```

Verify the velero backup using below command:

```bash
velero backup get
```

you should see the output similar to what you have seen for aws backup:

```
NAME                  STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
eks-wp-mysql-backup   Completed   0        0          2024-02-02 11:41:51 +0530 IST   29d       default            <none>

```

### Next Step
[Map the storage class of your EKS to AKS](map-storageclass-eks-aks.md)