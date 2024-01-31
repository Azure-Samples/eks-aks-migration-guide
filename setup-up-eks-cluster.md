### Some Pre-Requisites:

You need to have an AWS account. It cannot be the Starter Program since EKS is not supported there. Secondly, you must have a basic knowledge of AWS and Kubernetes. Third, you must have AWS CLI set up in your system with a dedicated profile allowing ADMIN Access so that it can directly use the EKS.

Although AWS CLI provides commands to manage EKS, but they are not efficient enough to perform complex tasks. Therefore, we are going to use another CLI built especially for EKS. You can download it from the GitHub link given below.
[**weaveworks/eksctl**
*eksctl is a simple CLI tool for creating clusters on EKS — Amazon’s new managed Kubernetes service for EC2. It is…*github.com](https://github.com/weaveworks/eksctl)

Apart from that, we need to have **kubectl** installed in our system too, for communicating with the Pods running on EKS. It is a managed service so everything will be managed by it except kubectl command because it is a client program, which will help us to connect with the pods.
[**Install and Set Up kubectl**
*The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters. You can use kubectl…*kubernetes.io](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### Starting the EKS Cluster

To start the EKS cluster, we need to set up a YAML file containing the infrastructure of the cluster. Information like the number of Worker Nodes, allowed EC2 instances, AWS key for connecting the instances with our local terminal and many more, are mentioned in this file.

After we write the desired infrastructure in our YAML file, we will have to execute the file with the EKSCTL CLI we have installed.

    eksctl create cluster -f cluster.yaml

This command will create the entire cluster in 1 click. The creation of the cluster would take a certain amount of time.

### Setting up the kubectl CLI

After the cluster is launched, we need to connect our system with the pods so that we can work on the cluster. Kubernetes has been installed in the instances already by EKS. Therefore to connect our kubectl with the Kubernetes on the instances, we need to update the KubeConfiguration file first. For this, we use the following command:

    aws eks update-kubeconfig  --name Ekscluster

We can check the connectivity with the command: kubectl cluster-info

For finding the number of nodes: kubectl get nodes

For finding the number of pods: kubectl get pods

To get detailed information of the instances on which the pods are running: kubectl get pods -o wide

Before we work, we need to create a namespace for our application in the K8s.

For that we use the following command: kubectl create namespace wp-mysql

Now we have to set it to be the default Namespace:

    kubectl config set-context --current --namespace=wp-mysql

For checking how many pods is running inside the namespace ‘**kube-system’** we have to execute : kubectl get pods -n kube-system

## Creating the Amazon EBS CSI driver IAM role

There are mainly 3 different types of storages available: File, Block and Object Storage. By default, EKS will set the storage system for the clusters as Block Storage and will be using EBS for that so we need to take few additional steps to install EBS driver to our cluster. Before intsalling the addon we will verify that our AWS IAM OpenID Connect (OIDC) provider exists for your cluster, run the following command: 

Determine whether you have an existing IAM OIDC provider for your cluster.

- View your cluster's OIDC provider URL.
```bash
aws eks describe-cluster --name eks-aks-k8s-cluster --query "cluster.identity.oidc.issuer" --output text
```
An example output is as follows.
`https://oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE`

Determine whether an IAM OIDC provider with your cluster's ID is already in your account.

```bash
aws iam list-open-id-connect-providers | grep OIDC_PROVIDER_ID
```
Note: Replace ID of the oidc provider with your OIDC ID. If you receive a No OpenIDConnect provider found in your account error, you must create an IAM OIDC provider.

- Create an IAM OIDC identity provider for your cluster with the following command

```bash
eksctl utils associate-iam-oidc-provider --cluster eks-aks-k8s-cluster --approve 
```

Create an IAM trust policy file, similar to the following example

```bash
cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/YOUR_OIDC ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>:aud": "sts.amazonaws.com",
          "oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF
```
Note: Replace YOUR_AWS_ACCOUNT_ID with your account ID. Replace YOUR_AWS_REGION with your Region. Replace your OIDC ID with the output from creating your IAM OIDC provider.

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
aws eks create-addon \
 --cluster-name eks-aks-k8s-cluster \
 --addon-name aws-ebs-csi-driver \
 --service-account-role-arn arn:aws:iam::
YOUR_AWS_ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole
```
Note: YOUR_AWS_ACCOUNT_ID with your account ID.

Verify that the EBS CSI driver installed successfully:

```bash
eksctl get addon --cluster eks-aks-k8s-cluster | grep ebs
```
A successfully installation returns the following output:
`aws-ebs-csi-driver    vx.xx.x-eksbuild.x    ACTIVE    0    arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/AmazonEKS_EBS_CSI_Driver`