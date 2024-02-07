## Setup Velero in EKS

#### Create an S3 Bucket to store backups

Velero uses S3 to store EKS backups in AWS. Here, we will create the S3 bucket to store EKS backup copy. To declare the unique S3 bucket name and appropriate AWS region as environment variables in a Linux or MacOS terminal, you can use the command:

```bash
BUCKET=aws-velero-bucket
REGION=us-east-2
aws s3 mb s3://$BUCKET --region $REGION
```

#### Create an IAM policy to grant Velero permissions

Velero performs a number of API calls to resources in EC2 and S3 to perform snapshots and save the backup to the S3 bucket. The following IAM policy will grant Velero the necessary permissions.

```bash
cat > velero_policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF
```

```bash
POLICY_NAME=VeleroAccessPolicy
aws iam create-policy --policy-name $POLICY_NAME --policy-document file://velero_policy.json
POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName=='$POLICY_NAME'].Arn' --output text)

```
Which returns an output similar to:

```bash
{
    "Policy": {
        "PolicyName": "VeleroAccessPolicy",
        "PolicyId": "ABCDEFGHIJKLMNOPQRST",
        "Arn": "arn:aws:iam::123456789012:policy/VeleroAccessPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "CreateDate": "2024-01-28T12:00:00Z",
        "UpdateDate": "2024-01-28T12:00:00Z"
    }
}
```
Copy the `ARN` and create a bash variable as shown below:

```bash
POLICY_ARN='<<Paste the ARN which you copied from the previous step here>>'

# or you can also get the ARN using below command
#POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName=='$POLICY_NAME'].Arn' --output text)
```

#### Create an IAM role and attach the trust relationship policy

```bash
EKS_CLUSTER_NAME=eks-aks-k8s-cluster
NAMESPACE=velero
SERVICE_ACCOUNT=velero
VELERO_ROLE=eks-velero-backup
OIDC_ID=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
```

```bash
cat > velero-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$ACCOUNT:oidc-provider/oidc.eks.$REGION.amazonaws.com/id/$OIDC_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.$REGION.amazonaws.com/id/$OIDC_ID:aud": "sts.amazonaws.com",
          "oidc.eks.$REGION.amazonaws.com/id/$OIDC_ID:sub": "system:serviceaccount:$NAMESPACE:$SERVICE_ACCOUNT"
        }
      }
    }
  ]
}          
EOF
```

```bash
aws iam create-role --role-name $VELERO_ROLE --assume-role-policy-document file://velero-trust-policy.json
aws iam attach-role-policy --role-name $VELERO_ROLE --policy-arn $POLICY_ARN
```

### Installing Velero server in EKS Cluster

Velero supports backing up and restoring Kubernetes volumes attached to pods from the file system of the volumes, called File System Backup (FSB shortly) or Pod Volume Backup. The data movement is fulfilled by using modules from free open-source backup tools [restic](https://github.com/restic/restic) and [kopia](https://github.com/kopia/kopia). Velero allows to take snapshots of persistent volumes as part of backups.

Velero `Node Agent` is a Kubernetes daemonset that hosts FSB modules, i.e., restic, kopia uploader & repository. Velero allows us to take snapshots of persistent volumes as part of our backups using one of the supported cloud providers’ block storage offerings. In our case it is Amazon EBS Volumes. 

**As in our case we need to backup persitent data attached to EBS volume we will set the `--deployNodeAgent=true` , `--snapshotsEnabled=true` and `--uploaderType="restic"` flags with velero server installation.**

Run the following command to install Velero in EKS cluster:

```bash
PRIMARY_CONTEXT=eks_backup_velero
aws eks --region $REGION update-kubeconfig --name $EKS_CLUSTER_NAME --alias $PRIMARY_CONTEXT
kubectl config use-context $PRIMARY_CONTEXT
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
```

```bash
cat > values.yaml <<EOF
configuration:
  backupStorageLocation:
    bucket: $BUCKET
  provider: aws
  volumeSnapshotLocation:
    config:
      region: $REGION
credentials:
  useSecret: false
initContainers:
- name: velero-plugin-for-aws
  image: velero/velero-plugin-for-aws:v1.0.0
  volumeMounts:
  - mountPath: /target
    name: plugins
serviceAccount:
  server:
    annotations:
      eks.amazonaws.com/role-arn: "arn:aws:iam::$ACCOUNT:role/$VELERO_ROLE"
EOF
```
```bash
helm install velero vmware-tanzu/velero -f values.yaml --namespace velero --set snapshotsEnabled=true --set deployNodeAgent=true --set uploaderType="restic"
```
With this command, velero gets the parameters from the BackupStorageLocation config to compose the URL to the backup storage.

To Verify velero server installation, running the follwoing command:

```bash
kubectl get pods –n velero

#Output:
NAME                      READY   STATUS    RESTARTS   AGE
node-agent-28lx7          1/1     Running   0          3d18h
node-agent-d78wq          1/1     Running   0          3d18h
node-agent-htrrw          1/1     Running   0          3d18h
velero-6686c4848f-nr487   1/1     Running   0          3d18h
```
Check if the Backup Storage Location is in “Available” state: 

```bash
velero backup-location get

#Output:
NAME      PROVIDER   BUCKET/PREFIX       PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   aws        aws-velero-bucket   Available   2024-02-02 11:52:54 +0530 IST   ReadWrite     true
```

### Installing the Velero CLI

Velero works by giving the cluster commands using CRDs. When you want to back up your cluster, you send a backup command to it using a backup CRD. Creating these by hand can be tricky, but the Velero team has made it easier with a user-friendly CLI. Today, we'll use the Velero CLI to create a backup for our EKS cluster.

Mac:
```
brew install velero
```

Linux:
```
wget https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
tar -xvf velero-v1.13.0-linux-amd64.tar.gz
mv velero /usr/local/bin/velero
```
Windows:
```
choco install velero
```
Installation instructions vary depending on your operating system. Follow the instructions to install Velero [here](https://velero.io/docs/v1.13/basic-install/#install-the-cli).

Now we will see how to back up the eks cluster. Velero creates one backup repo per namespace. For example, if we are backing up 2 namespaces, ns1 and ns2, using restic repository on AWS S3, the full backup repo path for namespace1 would be https://s3-us-east-1.amazonaws.com/bucket/restic/ns1 and for namespace2 would be https://s3-us-east-1.amazonaws.com/bucket/restic/ns2. 

Velero supports two approaches of discovering pod volumes that need to be backed up using FSB:

- Opt-in approach: Where every pod containing a volume to be backed up using FSB must be annotated with the volume’s name.
- Opt-out approach: Where all pod volumes are backed up using FSB, with the ability to opt-out any volumes that should not be backed up.

Since we need to backup only specific namespace which runs wordpress and MySQL pods volumes we will be using `opt-in approach` to backup pod volumes. To create a backup, we need to annotate each pod containing volume.

Run the following command for each pod that contains a volume to back up. Make sure the current kubectl context is set to eks cluster.

```bash
kubectl -n YOUR_POD_NAMESPACE annotate pod/YOUR_POD_NAME backup.velero.io/backup-volumes=YOUR_VOLUME_NAME_1,YOUR_VOLUME_NAME_2,...
```
Alternatively, if you want to backup all pod volumes without having to apply annotation on the pod when using file system backup set `--default-volumes-to-fs-backup` flag to backup command as shown below:

```bash
velero backup create BACKUP_NAME --include-namespaces wp-mysql  --default-volumes-to-fs-backup --wait
```
In our case I have used the below command to annotate the pods running our wordpress and mysql.

```
kubectl -n wp-mysql annotate pod/wordpress-79d68d56b9-z6dlw backup.velero.io/backup-volumes=wordpress-persistent-storage

kubectl -n wp-mysql annotate pod/wordpress-mysql-6b7b9b4c87-7vdlx backup.velero.io/backup-volumes=mysql-persistent-storage
```
Run the command below to create a backup of the wordpress application with mysql database running in wp-mysql namespace on eks cluster:

```bash
velero backup create eks-wp-mysql-backup --include-namespaces wp-mysql --wait
```
This will return a response similar to:

```
Backup request "eks-wp-mysql-backup" submitted successfully.
Waiting for backup to complete. You may safely press ctrl-c to stop waiting - your backup will continue in the background.
.............................................
Backup completed with status: Completed. You may check for more information using the commands `velero backup describe eks-wp-mysql-backup` and `velero backup logs eks-wp-mysql-backup`.
```
Validate a successful backup to check that the backup has been submitted successfully by running the command:

```bash
velero backup describe eks-wp-mysql-backup --details
```
the output of the above command looks similar to:

```bash
Name:         eks-wp-mysql-backup
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  velero.io/resource-timeout=10m0s
              velero.io/source-cluster-k8s-gitversion=v1.28.3
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=28

Phase:  Completed


Namespaces:
  Included:  wpmsqlnew
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  <none>

Or label selector:  <none>

Storage Location:  default

Velero-Native Snapshot PVs:  auto
Snapshot Move Data:          false
Data Mover:                  velero

TTL:  720h0m0s

CSISnapshotTimeout:    10m0s
ItemOperationTimeout:  4h0m0s

Hooks:  <none>

Backup Format Version:  1.1.0

Started:    2024-02-02 11:41:51 +0530 IST
Completed:  2024-02-02 11:42:36 +0530 IST

Expiration:  2024-03-03 11:41:51 +0530 IST

Total items to be backed up:  72
Items backed up:              72

Velero-Native Snapshots: <none included>

restic Backups (specify --details for more information):
  Completed:  2
```

To get the backup details run the below command:

```bash
velero backup get eks-wp-mysql-backup
```
you will see a similar response as:

```
NAME                  STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION    SELECTOR
eks-wp-mysql-backup   Completed   0        0          2024-01-28 13:20:34 +0530 IST   29d       default              <none>

```
To get the information about your pod volume backup, run the below command:

```bash
kubectl -n velero get podvolumebackups -l velero.io/backup-name=eks-wp-mysql-backup -o yaml
```
you will see a similar response as:

```bash
apiVersion: v1
items:
- apiVersion: velero.io/v1
  kind: PodVolumeBackup
  metadata:
    annotations:
      velero.io/pvc-name: mysql-pv-claim
    creationTimestamp: "2024-01-28 13:20:30Z"
    generateName: eks-wp-mysql-backup-
    generation: 1
    labels:
      velero.io/backup-name: eks-wp-mysql-backup
      velero.io/backup-uid: 6fee2294-6f20-4c88-8393-6307ab0ea30f
      velero.io/pvc-uid: d9e001a4-25be-4541-89ea-71775a643d94
    name: eks-wp-mysql-backup-csn7l
    namespace: velero
    ownerReferences:
    - apiVersion: velero.io/v1
      controller: true
      kind: Backup
      name: eks-wp-mysql-backup
      uid: 6fee2294-6f20-4c88-8393-6307ab0ea30f
    resourceVersion: "4242539"
    uid: 99f7cdcb-1aa5-44af-a1e1-8e784e05ba2d
  spec:
    backupStorageLocation: default
    node: ip-192-168-19-225.us-east-2.compute.internal
    pod:
      kind: Pod
      name: wordpress-79d68d56b9-9qjnc
      namespace: wp-mysql
      ...
      ...

```
To check the backup files created by Velero in the Amazon S3 bucket previously created:

```bash
aws s3 ls $BUCKET/backups/eks-wp-mysql-backup/
```

### Next Step
[Setting Up AKS Cluster](setup-aks-cluster.md)
