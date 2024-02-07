## Installing WordPress and MySQL in EKS cluster

Now, we are ready to install WordPress and Mysql in eks cluster. Copy the 3 files given in a folder `Wordpress-Mysql`.

The file `mysql-deployment.yml` contains information about the different settings to be applied to MySQL pod.

Similarly, the file `wordpress-deployment.yml` contains information about the different settings to be applied to WordPress pod.

At last, a `kustomization.yml` file to specify the order of execution of the files along with the secret keys.

Before we deploy our kubenetes resources, we need to create a namespace for applications.

Use the following command to create namespace: `kubectl create namespace wp-mysql`

Change the working directory to `Wordpress-Mysql` directory from cli and run the command `kubectl create -k . -n wp-mysql`, the scripts will be executed and create all the resources for deploying a WordPress site and a MySQL database.

Verify that the Pod is running by running the following command:
```bash
kubectl get pods -n wp-mysql
```
The response should be like this:

```bash
NAME                               READY   STATUS    RESTARTS   AGE
wordpress-79d68d56b9-9qjnc         1/1     Running   0          11m
wordpress-mysql-6b7b9b4c87-bzvl5   1/1     Running   0          11m
```

Verify that a PersistentVolume got dynamically provisioned by running the following command.

```bash
kubectl get pvc -n wp-mysql
```
The response should be like this:

```bash
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-1e879b20-5794-4258-b9be-64c9a9de6bca   20Gi       RWO            gp2            11m
wp-pv-claim      Bound    pvc-40345585-969d-4b37-91b0-eaca60b79661   20Gi       RWO            gp2            11m
```

Verify that the Secret exists by running the following command:

```bash
kubectl get secrets -n wp-mysql
```
The response should be like this:

```bash
NAME                    TYPE     DATA   AGE
mysql-pass-8d668bfdmt   Opaque   1      12m
```

Get the `External-IP` of wordpress service by running the below command and browse the wordpress site:

```bash
kubectl get services wordpress -n wp-mysql
```
The response should be like this:

```bash
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP                             PORT(S)        AGE
wordpress   LoadBalancer   10.0.206.193   xxxx-xx.us-east-2.elb.amazonaws.com     80:30569/TCP   13m
```

On visiting the `External-IP` URL, we will reach this page.

![](media/Wordpress-Install.png)

> [!WARNING]
> Do not leave your WordPress installation on this page. If another user finds it, they can set up a website on your instance and use it to serve malicious content.

After configuration and posting our first post, we will reach to this following page:

![](media/Wordpress-Landing-Page.png)

### Next Step
[Setup Velero in EKS and take bakup](setup-velero-backup-eks.md)
