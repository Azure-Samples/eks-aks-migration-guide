## Setup EKS Cluster

This guide helps you to create all of the required resources to get started with Amazon Elastic Kubernetes Service (Amazon EKS) using eksctl, a simple command line utility for creating and managing Kubernetes clusters on Amazon EKS. At the end of this tutorial, you will have a running Amazon EKS cluster that you can deploy applications to.

### Prerequisites:
Before you start this tutorial, check you have these prerequisites in place.

- An AWS account and IAM credentials.
- AWS CLI set up - A command line tool for working with AWS resources. For more information, see [Install or update the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- `kubectl` – A command line tool for working with Kubernetes clusters. For more information, see [Installing or updating kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).
- `eksctl` – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see [Installation](https://eksctl.io/installation) in the `eksctl` documentation.
- `Required IAM permissions` – The IAM security principal that you're using must have permissions to work with Amazon EKS IAM roles, service linked roles, AWS CloudFormation, a VPC, and related resources. For more information, see [Actions, resources, and condition keys for Amazon Elastic Container Service for Kubernetes](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonelastickubernetesservice.html) and [Using service-linked roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html) in the IAM User Guide. You must complete all steps in this guide as the same user. To check the current user, run the following command:

```bash
aws sts get-caller-identity
```

### Create Amazon EKS cluster and nodes

To create the EKS cluster, set up a YAML file containing the infrastructure of the cluster like Worker Nodes, EC2 instances. All this information are mentioned in file `cluster.yaml` in this repo.

Execute the below command for creating the cluster:

```bash
eksctl create cluster -f cluster.yaml
```
The creation of the cluster would take a certain amount of time.

### Connect to the cluster

After cluster creation is complete, view the kubenetes resources created in EKS using the following command.

```bash
PRIMARY_CONTEXT=eks_backup_velero
REGION=us-east-1
EKS_CLUSTER_NAME=eks-aks-k8s-cluster
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
```
> [!NOTE]  
> Setting the above variables to use in subsequent steps.
```bash
aws eks --region $REGION update-kubeconfig --name $EKS_CLUSTER_NAME --alias $PRIMARY_CONTEXT
```
Check the cluster connectivity with the command: `kubectl cluster-info`

For finding the number of nodes: `kubectl get nodes`

For finding the number of pods: `kubectl get pods`

## Creating the Amazon EBS CSI driver IAM role

There are mainly 3 different types of storages available: File, Block and Object Storage. By default, EKS will set the storage system for the clusters as Block Storage and will be using EBS. Few additional steps are reqquired to install EBS driver to EKS cluster. Before installing the addon, verify AWS IAM OpenID Connect (OIDC) provider exists on the cluster, run the following command: 

To check the existing IAM OIDC provider for cluster.

- View cluster's OIDC provider URL.
```bash
OIDC_ID=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
Check whether an IAM OIDC provider exists with the cluster's ID.

```bash
aws iam list-open-id-connect-providers | grep $OIDC_ID
```
**Note:** If you receive a message similar to "No OpenIDConnect provider found". create an IAM OIDC provider.

- Create an IAM OIDC identity provider for cluster with the following command

```bash
eksctl utils associate-iam-oidc-provider --cluster $EKS_CLUSTER_NAME --approve 
```
### Creating a IAM trust policy

Create an IAM trust policy file, using the below template:

```bash
cat <<EOF > trust-policy.json
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
          "oidc.eks.$REGION.amazonaws.com/id/$OIDC_ID:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF
```
Create an IAM role named Amazon_EBS_CSI_Driver:

```bash
aws iam create-role \
 --role-name AmazonEKS_EBS_CSI_Driver \
 --assume-role-policy-document file://"trust-policy.json"
```
Attach the AWS managed IAM policy for the EBS CSI Driver to the IAM role that we created:

```bash
aws iam attach-role-policy \
--policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--role-name AmazonEKS_EBS_CSI_Driver
```
Deploy the Amazon EBS CSI driver.
```bash
aws eks create-addon --cluster-name $EKS_CLUSTER_NAME --addon-name aws-ebs-csi-driver --service-account-role-arn arn:aws:iam::$ACCOUNT:role/AmazonEKS_EBS_CSI_DriverRole
```
Verify that the EBS CSI driver installed successfully:

```bash
eksctl get addon --cluster $EKS_CLUSTER_NAME | grep ebs
```
A successfully installation returns the following output:
```
aws-ebs-csi-driver    vx.xx.x-eksbuild.x    ACTIVE    0    arn:aws:iam::$ACCOUNT:role/AmazonEKS_EBS_CSI_Driver
```

### Next Step
[Deploy WordPress and MySQL Application](deploy-wordpress-mysql-eks.md)
