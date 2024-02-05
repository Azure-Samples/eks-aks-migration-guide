# Abstract:
As cloud-native technologies continue to evolve, organizations often find themselves seeking ways to optimize their infrastructure, reduce costs, or leverage specific features offered by different cloud providers. Migrating Kubernetes workloads between cloud environments is a common scenario, but it can be complex and daunting without the right tools and strategies in place. In this article, we show a comprehensive guide to migrating from `Amazon EKS` to `Azure AKS` using `Velero`, a powerful open-source tool for Kubernetes cluster backup, restore, and migration.

## Introduction:
In recent years, Kubernetes has emerged as the de facto standard for container orchestration, empowering organizations to deploy, scale, and manage containerized applications with ease. Managed Kubernetes services, such as Amazon EKS and Azure AKS, offer a simplified approach to Kubernetes cluster management, allowing businesses to focus on application development and innovation rather than infrastructure management.

However, shifting workloads between cloud providers is not always straightforward. Challenges such as data migration, resource compatibility, and application downtime must be carefully addressed to ensure a seamless transition. Velero, formerly known as Heptio Ark, addresses these challenges by providing robust backup and migration capabilities for Kubernetes clusters.

## Understanding Amazon EKS and Azure AKS:
Amazon EKS and Azure AKS are fully managed Kubernetes services offered by Amazon Web Services (AWS) and Microsoft Azure, respectively. These services abstract the complexities of Kubernetes cluster management, including infrastructure provisioning, scaling, and maintenance, while providing high availability and reliability.

### Amazon EKS (Elastic Kubernetes Service):
Amazon EKS enables users to deploy, manage, and scale containerized applications using Kubernetes on AWS. It integrates seamlessly with other AWS services, offering features such as auto-scaling, managed node groups, and AWS Fargate integration. For more details [See](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)

### Azure AKS (Azure Kubernetes Service):
Azure AKS provides a similar managed Kubernetes experience on the Azure cloud platform. It offers features such as automatic updates, horizontal scaling, and integration with Azure Active Directory and Azure Monitor. For more details [See](https://learn.microsoft.com/en-in/azure/aks/).

## Introduction to Velero:
Velero is an open-source tool for Kubernetes cluster backup, restore, and migration. Developed by Heptio, now part of VMware, Velero simplifies the process of protecting Kubernetes applications and data, whether in single or multi-cluster environments.

Key features of Velero include:

1. __Backup and Restore__: Velero enables users to capture snapshots of entire Kubernetes clusters, including namespaces, resources, and persistent volumes. These backups can be restored to the same cluster or migrated to another environment.

2. __Plugin Architecture__: Velero supports various cloud providers and storage solutions through its plugin architecture. Users can configure Velero to work with cloud-specific APIs and storage providers, ensuring compatibility across different environments.

3. __Schedule and Retention Policies__: Velero allows users to define backup schedules and retention policies, ensuring that backups are performed regularly and retained for the desired duration.

for more details about velero, [See](https://velero.io/docs/v1.13/)

## Migration Process Overview:
Migrating from Amazon EKS to Azure AKS using Velero involves several distinct steps, each designed to ensure a smooth and seamless transition. The migration process can be summarized as follows:

__Preparation__: Prepare both the source (Amazon EKS) and target (Azure AKS) environments, ensuring that necessary permissions, credentials, and configurations are in place.

__Backup__: Use Velero to create a backup of the Amazon EKS cluster, capturing all relevant resources and data.

__Storage Transfer__: Transfer the Velero backup data from the source cloud provider (AWS S3 bucket) to the target cloud provider (Azure Storage account).

__Restore__: Use Velero to restore the backup data to the Azure AKS cluster, recreating the Kubernetes resources and configurations.

In this article I am going to explain each steps in details starting from the creation of EKS cluster, deploying WordPress and MySQL applications to eks cluster, setting up Velero in EKS for taking backups, setting up AKS cluster, setting up Velero in AKS for restoring backups to AKS cluster.

__Step 1__: [Creating EKS Cluster with EBS Driver](setup-eks-cluster.md)
__Step 2__: [Deploy WordPress and MySQL Application](deploy-wordpress-mysql-eks.md)
__Step 3__: [Setup Velero in EKS and take bakup](setup-velero-backup-eks.md)
__Step 4__: [Setting Up AKS Cluster](setup-aks-cluster.md)
__Step 5__: [Setup Velero in AKS with blob storage](setup-velero-aks.md)
__Step 6__: [Transfer Backup Data from AWS S3 Bucket and Restore in Azure blob storage using AzCopy](copy-data-using-Azcopy.md)
__Step 7__: [Map the storage class of your EKS to AKS](map-storageclass-eks-aks.md)
__Step 8__: [Restore the backup to AKS cluster](restore-aks-cluster.md)

