# Deploy JFrog Artifactory on a Kubernetes cluster

These are some Kubernetes manifests to deploy a JFrog Artifactory on a Kubernetes cluster

## On this page
- Pre-requisites
- Setup a NFS Server
- Setup NFS Clients
- Dynamic NFS provisioning
- Deploying a JFrog Artifactory instance
- Setup an Ingress Controller
- Deploy an Ingress object
- Setup Metallb load balancer
- Scaling the Jira Software Data Center cluster

## Pre-requisites
- You need a Kubernetes cluster up-and-running in order to run this example.
- You need a shared storage. For this example, we will use an external server (external to the Kubernetes cluster, I mean) and NFS protocol. In the following sections we’re going to describe how to configure this.
- You need to setup a load balancer to provision an external IP to make accessible the application from outside of the Kubernetes cluster.

## Setup a NFS Server
I will consider that we’re setting up the NFS Server on CentOS Stream 9, to stay consistent with the other servers.

Procedure and commands may be different from other Linux distributions. 

Let’s install the packages on the server we’re using like storage unit

```
$ sudo yum install nfs-utils
``` 

Now we create the folder that will be shared by NFS:
```
$ sudo mkdir /var/nfsshare
``` 

We change the permissions of the folder as follows:
```
$ sudo chmod -R 755 /var/nfsshare
$ sudo chown -R nobody:nobody /var/nfsshare
``` 

Next, we need to start the services and enable them to be started at boot time:
```
$ sudo systemctl enable rpcbind
$ sudo systemctl enable nfs-server
$ sudo systemctl enable nfs-lock
$ sudo systemctl enable nfs-idmap
$ sudo systemctl start rpcbind
$ sudo systemctl start nfs-server
$ sudo systemctl start nfs-lock
$ sudo systemctl start nfs-idmap
``` 

Now, we share the NFS folder over the network as follows:
```
$ sudo vi /etc/exports
/etc/exports

# '*' enables any IP to access to the NFS folder
/var/nfsshare    *(rw,sync,no_root_squash,no_all_squash)
``` 

We restart the NFS service:
```
$ sudo systemctl restart nfs-server
``` 

Finally, it’s better if we stop and disable the firewall (not recommended in production environments):
```
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
```

## Setup NFS Clients
The NFS clients will be the Kubernetes cluster worker nodes. Thus, on both servers, worker1 and worker2, we have to install the nfs-utils package, as we did on the NFS server:
```
$ sudo yum install nfs-utils
```

## Dynamic NFS provisioning
In this example, we’re going to provision NFS resources in both manually and dynamically ways. For the dynamic NFS provisioning, we need to deploy a [nfs provisioner](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) on our Kubernetes cluster. 

There are some object to be created before deploying the nfs provisioner. First, we create a [Service Account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) and some roles and role bindings required by Kubernetes:

**rbac.yaml**
```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-pod-provisioner-sa
---
kind: ClusterRole # Role of kubernetes
apiVersion: rbac.authorization.k8s.io/v1 # auth API
metadata:
  name: nfs-provisioner-clusterRole
rules:
  - apiGroups: [""] # rules on persistentvolumes
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-provisioner-rolebinding
subjects:
  - kind: ServiceAccount
    name: nfs-pod-provisioner-sa # defined on top of file
    namespace: default
roleRef: # binding cluster role to service account
  kind: ClusterRole
  name: nfs-provisioner-clusterRole # name defined in clusterRole
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-pod-provisioner-otherRoles
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-pod-provisioner-otherRoles
subjects:
  - kind: ServiceAccount
    name: nfs-pod-provisioner-sa # same as top of the file
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: nfs-pod-provisioner-otherRoles
  apiGroup: rbac.authorization.k8s.io
```
```
[kubi@master ~]$ kubectl apply -f rbac.yaml
serviceaccount/nfs-pod-provisioner-sa created
clusterrole.rbac.authorization.k8s.io/nfs-provisioner-clusterRole created
clusterrolebinding.rbac.authorization.k8s.io/nfs-provisioner-rolebinding created
role.rbac.authorization.k8s.io/nfs-pod-provisioner-otherRoles created
rolebinding.rbac.authorization.k8s.io/nfs-pod-provisioner-otherRoles created
```

Second, we deploy a [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/):

**nfs-class.yaml**
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storageclass # IMPORTANT pvc needs to mention this name
provisioner: nfs-pod-provisioner # name can be anything
parameters:
  archiveOnDelete: "false"
```
```
kubi@master ~]$ kubectl apply -f nfs-class.yaml
storageclass.storage.k8s.io/nfs-storageclass created
```

And finally, we can proceed with the deploy of the nfs provisioner:

**nfs-provisioner-deployment.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-pod-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-pod-provisioner
  template:
    metadata:
      labels:
        app: nfs-pod-provisioner
    spec:
      serviceAccountName: nfs-pod-provisioner-sa # name of service account created in rbac.yaml
      containers:
        - name: nfs-pod-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-provisioner-v
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME # do not change
              value: nfs-pod-provisioner # SAME AS PROVISONER NAME VALUE IN STORAGECLASS
            - name: NFS_SERVER # do not change
              value: 10.0.2.15 # Ip of the NFS SERVER
            - name: NFS_PATH # do not change
              value: /var/nfsshare # path to nfs directory setup
      volumes:
       - name: nfs-provisioner-v # same as volumeMounts name
         nfs:
           server: 10.0.2.15 # Ip of the NFS SERVER
           path: /var/nfsshare # path to nfs directory setup
```
```
[kubi@master ~]$ kubectl apply -f nfs-provisioner-deployment.yaml 
deployment.apps/nfs-pod-provisioner created

[kubi@master  ~]$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
nfs-pod-provisioner-54499dd8ff-x9lt7   1/1     Running   0          34s
```

## Deploying PostgreSQL instance
For this example, we’re going to use PostgreSQL as database. We just need to setup a single instance. For that, we’ll follow the steps below:

We deploy a ConfigMap with the configuration of our instance:
```
[kubi@master ~]$ kubectl apply -f postgres-config.yaml 
configmap/postgres-config created
``` 

We create a PersistentVolume manually:
```
[kubi@master ~]$ kubectl apply -f postgres-pv-nfs.yaml
persistentvolume/postgres-pv-nfs-volume created

[kubi@master ~]$ kubectl get pv
NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                          STORAGECLASS       REASON   AGE
postgres-pv-nfs-volume   2Gi        RWO            Retain           Available   default/postgres-persistent-nfs-volume-claim   nfs-storageclass            8s
``` 

We need a PersistentVolumeClaim. We create it manually:
```
[kubi@master ~]$ kubectl apply -f postgres-pvc-nfs.yaml 
persistentvolumeclaim/postgres-persistent-nfs-volume-claim created

[kubi@master ~]$ kubectl get pvc
NAME                                   STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-persistent-nfs-volume-claim   Bound    postgres-pv-nfs-volume   2Gi        RWO            nfs-storageclass  5s
``` 

Before to apply the Postgres deployment, we need to create manually the folder /var/nfsshare/pgdata in the NFS server:
```
[cmoya@storage ~]$ sudo mkdir /var/nfsshare/pgdata
``` 

Now, we apply the Postgres deployment manifest:
```
[kubi@master ~]$ kubectl apply -f postgres-deployment-nfs.yaml 
deployment.apps/postgres created

[kubi@master ~]$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
nfs-pod-provisioner-54499dd8ff-x9lt7   1/1     Running   0          26m
postgres-6997bbb746-v6cmp              1/1     Running   0          5m18s
``` 

Let’s check that everything went well:
```
[kubi@master ~]$ kubectl logs postgres-6997bbb746-v6cmp
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.
The database cluster will be initialized with locale "en_US.utf8".
The default database encodin:wq!g has accordingly been set to "UTF8".
The default text search configuration will be set to "english".
Data page checksums are disabled.
fixing permissions on existing directory /var/lib/postgresql/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default timezone ... Etc/UTC
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.
syncing data to disk ... ok
Success. You can now start the database server using:
    pg_ctl -D /var/lib/postgresql/data -l logfile start
waiting for server to start....LOG:  database system was shut down at 2020-09-02 13:56:38 UTC
LOG:  MultiXact member wraparound protections are now enabled
LOG:  database system is ready to accept connections
LOG:  autovacuum launcher started
 done
server started
CREATE DATABASE
/usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*
waiting for server to shut down...LOG:  received fast shutdown request
LOG:  aborting any active transactions
LOG:  autovacuum launcher shutting down
.LOG:  shutting down
LOG:  database system is shut down
 done
server stopped
PostgreSQL init process complete; ready for start up.
LOG:  database system was shut down at 2020-09-02 13:57:02 UTC
LOG:  MultiXact member wraparound protections are now enabled
LOG:  database system is ready to accept connections
LOG:  autovacuum launcher started
``` 

Finally, we create a Service to make the database accessible inside the cluster:
```
[kubi@master ~]$ kubectl apply -f postgres-svc.yaml 
service/postgres created

[kubi@master ~]$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    171m
postgres     ClusterIP   10.96.123.91   <none>        5432/TCP   8s
``` 

## Deploying a Jira Software Data Center cluster
To setup a Jira Software Data Center cluster in Kubernetes, we will require the following objects:

- A ConfigMap, that we’ll use to configure most of the properties of our Jira Software cluster.

- A PersistentVolumeClaim, that we’ll create manually because we just need one shared volume across the cluster.

- A Statefulset, that is a specific way to deploy an application cluster, offering advanced features not present in a normal Deployment object

- A Service, to make accessible the pods inside the Kubernetes cluster

Let’s start:
```
[kubi@master ~]$ kubectl apply -f jira-dc-config.yaml 
configmap/jira-dc-config created

[kubi@master ~]$ kubectl get configmaps
NAME              DATA   AGE
jira-dc-config    10     11s
postgres-config   3      37m

[kubi@master ~]$ kubectl apply -f jira-dc-shared-pvc.yaml 
persistentvolumeclaim/jira-dc-shared-pvc created

[kubi@master ~]$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
jira-dc-shared-pvc                     Bound    pvc-bb8dfae3-b15b-435d-9781-93a99bf8b4e7   1G         RWX            nfs-storageclass   9s
postgres-persistent-nfs-volume-claim   Bound    postgres-pv-nfs-volume                     10Gi       RWO                               32m

[kubi@master ~]$ kubectl apply -f jira-dc-statefulset.yaml 
statefulset.apps/jira-dc created

[kubi@master ~]$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
jira-dc-0                              1/1     Running   0          4m3s
nfs-pod-provisioner-54499dd8ff-x9lt7   1/1     Running   0          51m
postgres-6997bbb746-v6cmp              1/1     Running   0          30m

kubi@master ~]$ kubectl apply -f jira-dc-svc.yaml 
service/jira-dc created

[kubi@master ~]$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                               AGE
jira-dc      ClusterIP   None           <none>        80/TCP,8888/TCP,40001/TCP,40011/TCP   6s
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP                               3h14m
postgres     ClusterIP   10.96.123.91   <none>        5432/TCP                              23m
```

## Setup an Ingress Controller
As per [Kubernetes official documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/), an **Ingress** object may provide load balancing, SSL termination and name-based virtual hosting. We require this kind of object to load balance our Jira Software Data Center multi-node deployment.

You must have an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers) to satisfy an Ingress. For this example, we’re using the [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx/blob/master/README.md) officially supported by Kubernetes. We follow the instructions to install it in a [Bare-metal scenario](https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal):
```
[kubi@master ~]$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.0/deploy/static/provider/baremetal/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created

[kubi@master ~]$ kubectl get services -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.100.176.3    <none>        80:32400/TCP,443:32320/TCP   37s
ingress-nginx-controller-admission   ClusterIP   10.96.122.129   <none>        443/TCP                      38s
```

By default, this installation works in “NopePort” mode. We need to change that to “LoadBalancer”, because we plan to use a separate LoadBalancer who will be responsible to allocate external IPs to our cluster.
```
[kubi@master ~]$ kubectl edit service ingress-nginx-controller -n ingress-nginx
service/ingress-nginx-controller edited

  # We have to look for this entries and modify "type" and save changes
  
  sessionAffinity: None
  type: LoadBalancer # Previous value was NodePort
```
```
kubi@master ~]$ kubectl get services -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.176.3    <pending>     80:32400/TCP,443:32320/TCP   2m27s
ingress-nginx-controller-admission   ClusterIP      10.96.122.129   <none>        443/TCP                      2m28s
Deploy an Ingress object
```
```
[kubi@master ~]$ kubectl apply -f jira-dc-ingress.yaml 
ingress.networking.k8s.io/jira-dc-ingress created

[kubi@master ~]$ kubectl get ingress
NAME              CLASS   HOSTS   ADDRESS   PORTS   AGE
jira-dc-ingress   nginx   *                 80      16s
```

## Setup Metallb load balancer
We follow the instructions to Install Metallb on Kubernetes cluster described in the [MetalLB official documentation](https://metallb.universe.tf/installation/) 

After having followed the installation process, if we have a look at the ingress services, we should already see that an external IP has been assigned to the controller:
```
[kubi@master ~]$ kubectl get services -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.101.80.192   10.0.2.25     80:31742/TCP,443:31578/TCP   18m
ingress-nginx-controller-admission   ClusterIP      10.109.10.92    <none>        443/TCP                      18m
```

The IP address that appears in the EXTERNAL-IP section of the ingress-nginx-controller is the one through which we can access to our Jira Software Data Center application. 

We have to connect to the Jira application and finish the setup process before scaling the cluster from 1 node to 2

## Scaling the Jira Software Data Center cluster
For now, we have just 1 node in the cluster:
```
[kubi@master ~]$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
jira-dc-0                              1/1     Running   0          7m38s
nfs-pod-provisioner-54499dd8ff-4qmkt   1/1     Running   1          19d
postgres-6997bbb746-lf5nd              1/1     Running   1          19d
```

To scale it, what we have to do is scaling the statefulset previously deployed:
```
[kubi@master ~]$ kubectl scale statefulset jira-dc --replicas=2
statefulset.apps/jira-dc scaled

[kubi@master ~]$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
jira-dc-0                              1/1     Running   0          9m47s
jira-dc-1                              1/1     Running   0          27s
nfs-pod-provisioner-54499dd8ff-4qmkt   1/1     Running   1          19d
postgres-6997bbb746-lf5nd              1/1     Running   1          19d
```
