## Installing WordPress and MySQL

Now, we are ready to install WordPress and Mysql in our cluster. For that, we need to copy the 3 files given in a folder.

This file contains information about the different settings to be applied to our MySQL pod.

Similarly, this file contains information about the different settings to be applied to our WordPress pod.

At last, we create a **Kustomization file **to specify the order of execution of the files along with the secret keys.

After executing the scripts using the command `kubectl apply -k ./` in a folder Wordpress-Mysql, we can build the infrastructure.

Verify that the Pod is running by running the following command:
```bash
kubectl get pods
```
The response should be like this:

```bash
NAME                               READY   STATUS    RESTARTS      AGE
wordpress-79d68d56b9-z6dlw         1/1     Running   2 (41h ago)   41h
wordpress-mysql-6b7b9b4c87-7vdlx   1/1     Running   0             41h
```

Verify that a PersistentVolume got dynamically provisioned.

```bash
kubectl get pvc
```
The response should be like this:

```bash
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-f58ee046-1ae6-4c63-8c1e-c43148d722aa   20Gi       RWO            default        41h
wp-pv-claim      Bound    pvc-bfb68065-3096-4864-ad9f-67b92d3d774f   20Gi       RWO            default        41h
```

Verify that the Secret exists by running the following command:

```bash
kubectl get secrets
```
The response should be like this:

```bash
NAME                    TYPE     DATA   AGE
mysql-pass-8d668bfdmt   Opaque   1      42h
```

Get the `External-IP` of our wordpress service by running the below command and browse your wordpress site:

```bash
kubectl get services wordpress
```
The response should be like this:

```bash
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer   10.0.206.193   X.XXX.X.XX   80:30569/TCP   41h
```

On visiting the `External-IP`, we will reach this page.

![](media/Wordpress-Install.png)

After configuration and posting our first post, we will reach to this following page:

![](media/Wordpress-Landing-Page.png)
