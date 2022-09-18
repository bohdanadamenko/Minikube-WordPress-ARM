# Introduction

Wordpress setup in a minikube Kubernetes cluster for ARM(M1), using the folowing components:

- nginx reverse proxy (2 replicas)

- Wordpress (2 replicas)

- MySQL database

- Redis cache


# Setup 

```console

$ kubectl create secret generic mysql-secrets --from-literal=rootpw=secretpw
secret "mysql-secrets" created

$ kubectl get secrets
NAME            TYPE     DATA   AGE
mysql-secrets   Opaque   1      23s

# your password = "secretpw"

$ kubectl create -f mysql_vol.yaml
persistentvolumeclaim "mysql-volume-claim" created

$ kubectl create -f mysql.yaml
# form ARM we use MySQL image: biarms/mysql:5.7
deployment.apps "mysql-deployment" created
service "mysql-service" created


# Create WordPress MySQL DB

$ kubectl get po
NAME                                    READY   STATUS              RESTARTS   AGE
mysql-deployment-7678c5d844-878vd       1/1     Running             0          7m7s

$ kubectl exec -it mysql-deployment-7678c5d844-878vd bash
$ mysql -p
Enter password: 
$ CREATE DATABASE wordpress;
Query OK, 1 row affected (0.00 sec)


$ kubectl create -f redis_vol.yaml 
persistentvolumeclaim "redis-volume-claim" created

$ kubectl create -f redis.yaml 
deployment.apps/redis-deployment created
service/redis-service created


$ kubectl create -f nfs_server_vol.yaml
persistentvolumeclaim/nfs-server-volume-claim created

$ kubectl create -f nfs_server.yaml 
deployment.apps "nfs-deployment" created
service "nfs-server" created
$ kubectl get service nfs-service
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
nfs-service   ClusterIP   10.111.27.77   <none>        2049/TCP,20048/TCP,111/TCP   67s

change nfs-server IP in nfs_pv.yaml

$ kubectl create -f nfs_pv.yaml 
persistentvolume "nfs-volume" created

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES  RECLAIM POLICY   STATUS   CLAIM                             STORAGECLASS REASON  
nfs-volume                                 1Gi        RWX           Retain     Bound    default/nfs-volume-claim                                67s
pvc-5b4a458e-0ba9-4473-b626-2923bb31f577   1Gi        RWO           Delete     Bound    default/nfs-server-volume-claim   standard              5m19s
pvc-91f122ec-a792-458b-9f48-182011ea0fe8   2Gi        RWO           Delete     Bound    default/mysql-volume-claim        standard              7m32s
pvc-9f5f1052-267e-48f4-99b9-5549c4649ada   1Gi        RWO           Delete     Bound    default/redis-volume-claim        standard              7m7s

$ kubectl create -f nfs_pvc.yaml 
persistentvolumeclaim "nfs-volume-claim" created

$ kubectl get pvc
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-volume-claim        Bound    pvc-91f122ec-a792-458b-9f48-182011ea0fe8   2Gi        RWO            standard       7m40s
nfs-server-volume-claim   Bound    pvc-5b4a458e-0ba9-4473-b626-2923bb31f577   1Gi        RWO            standard       5m27s
nfs-volume-claim          Bound    nfs-volume                                 1Gi        RWX                           66s
redis-volume-claim        Bound    pvc-9f5f1052-267e-48f4-99b9-5549c4649ada   1Gi        RWO            standard       7m15s


$ kubectl create -f wordpress.yaml 
deployment.apps "wordpress-deployment" created
service "wordpress-service" created


$ kubectl create -f nginx.yaml 
configmap "nginx-configmap" created
deployment.apps "nginx-deployment" created
service "nginx-service" created


$ kubectl get po
NAME                                    READY   STATUS    RESTARTS   AGE
mysql-deployment-7678c5d844-878vd       1/1     Running   0          7m19s
nfs-deployment-7cbdc455dd-29k92         1/1     Running   0          4m11s
nginx-deployment-7fc48bf55b-h25ht       1/1     Running   0          18s
nginx-deployment-7fc48bf55b-m9244       1/1     Running   0          18s
redis-deployment-5f98f49989-lfmkg       1/1     Running   0          6m56s
wordpress-deployment-7c99b5d55d-gzzxc   1/1     Running   0          26s
wordpress-deployment-7c99b5d55d-vb5v8   1/1     Running   0          26s


$ kubectl get services
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
kubernetes          ClusterIP      10.96.0.1        <none>        443/TCP                      22m
mysql-service       ClusterIP      10.106.19.120    <none>        3306/TCP                     12m
nfs-service         ClusterIP      10.111.27.77     <none>        2049/TCP,20048/TCP,111/TCP   9m31s
nginx-service       LoadBalancer   10.96.126.227    <pending>     80:31611/TCP                 5m38s
redis-service       ClusterIP      10.101.146.135   <none>        6379/TCP                     12m
wordpress-service   ClusterIP      10.110.71.239    <none>        80/TCP                       5m46s


Connect to LoadBalancer services, create minikube tunnel:
$ minikube tunnel


$ curl -L -i 127.0.0.1

HTTP/1.1 302 Found
Server: nginx/1.23.1
Date: Mon, 19 Sep 2022 07:37:28 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 0
Connection: keep-alive
X-Powered-By: PHP/7.4.30
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0
X-Redirect-By: WordPress
Location: http://127.0.0.1/wp-admin/install.php
	...
```
