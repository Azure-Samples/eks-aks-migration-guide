## Setup Velero for EKS Backup

### Some Pre-Requisites:
AWS account with permissions to create and manage service accounts
EKS cluster [see](Setting-up-eks-cluster.md)
S3 bucket for backup storage
eksctl v0.167.0 or higher See [Installing or upgrading eksctl](https://eksctl.io/installation/).
AWS CLI version 2
kubectl, Helm v3

#### Storage Bucket & Policy Setup

Create an S3 Bucket to store backups

Velero uses S3 to store EKS backups when running in AWS. Here we’ll create the S3 bucket that will store our EKS backup copy.To declare the unique S3 bucket name and appropriate AWS region as environment variables in a Linux or MacOS terminal, you can use the command:

Replace <BUCKETNAME> and <REGION> with your own values below.
```bash
BUCKET=<BUCKETNAME>
REGION=us-east-1
aws s3 mb s3://$BUCKET --region $REGION
```

Create an IAM policy to grant Velero permissions

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

Create an IAM role and attach the trust relationship policy
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

Run the following command to install Velero in our EKS cluster:

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
We can check that the Velero server was successfully installed by running this command:

```bash
kubectl get pods –n velero
```
Check if the Backup Storage Location is in “Available” state: 

```bash
velero backup-location get

NAME      PROVIDER   BUCKET/PREFIX       PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
default   aws        aws-velero-bucket   Available   2024-01-31 12:57:34 +0530 IST   ReadWrite     true
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

Now we will show see how to back up the eks cluster. Make sure the current kubectl context is set to eks cluster.Run the command below to create a backup of the wordpress application with mysql database running in wp-mysql namespace on eks cluster:

```bash
velero backup create eks-wp-mysql-backup --include-namespaces wp-mysql --default-volumes-to-fs-backup --wait
```
This will return a response similar to:

```
Backup request "eks-wp-mysql-backup" submitted successfully.
Run `velero backup describe eks-wp-mysql-backup` or `velero backup logs eks-wp-mysql-backup` for more details.
```
Validate a successful backup to check that the backup has been submitted successfully by running the command:

```bash
velero backup describe eks-wp-mysql-backup
```
Look for the field `Phase:` in the output of the command. If the current `Phase` is `InProgress`, then wait a few moments and try again until you see the `Phase: Completed`. You can see additional details of the backup, including information such as the start time and completion time, along with the number of items backed up.

To get the backup details run the below command:

```bash
velero backup get eks-wp-mysql-backup
```
which should show a similar response as:

```
NAME                  STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION    SELECTOR
eks-wp-mysql-backup   Completed   0        0          2024-01-28 13:20:34 +0530 IST   29d       default              <none>

```
We can also see the backup files created by Velero in the Amazon S3 bucket we previously created:

```bash
aws s3 ls $BUCKET/backups/eks-wp-mysql-backup/
```
