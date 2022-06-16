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

There are some object to be created before deploying the nfs provisioner. Let's create previously a dedicated namespace

```
[kubi@master ~]$ kubectl apply -f rbac.yaml --namespace nfs
```

Then, we create a [Service Account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) and some roles and role bindings required by Kubernetes:

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
[kubi@master ~]$ kubectl apply -f rbac.yaml --namespace nfs
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
kubi@master ~]$ kubectl apply -f nfs-class.yaml --namespace nfs
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
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-provisioner-v
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME # do not change
              value: nfs-pod-provisioner # SAME AS PROVISONER NAME VALUE IN STORAGECLASS
            - name: NFS_SERVER # do not change
              value: 10.186.0.12 # Ip of the NFS SERVER
            - name: NFS_PATH # do not change
              value: /var/nfsshare # path to nfs directory setup
      volumes:
       - name: nfs-provisioner-v # same as volumemounts name
         nfs:
           server: 10.186.0.12 # Ip of the NFS SERVER
           path: /var/nfsshare # path to nfs directory setup
```
```
[kubi@master ~]$ kubectl apply -f nfs-provisioner-deployment.yaml --namespace nfs
deployment.apps/nfs-pod-provisioner created

[kubi@master  ~]$ kubectl get pods --namespace nfs
NAME                                   READY   STATUS    RESTARTS   AGE
nfs-pod-provisioner-6fd58ddc6d-nspt8   1/1     Running   0          27s
```

## Deploying a JFrog Artifactory instance
To setup a JFrog Artifactory instance in Kubernetes, we will require the following objects:

- 4 Secrets:
-- artifactory-access-config
-- artifactory-binarystore
-- artifactory
-- artifactory-systemyaml

- 2 ConfigMaps:
-- artifactory-migration-scripts
-- artifactory-installer-info

- 2 PersistentVolumeClaims:
-- artifactory-backup-pvc
-- artifactory-data-pvc

- 1 Stafefulset:
-- artifactory

- 1 Service, to make accessible the artifactory pod inside the Kubernetes cluster
-- artifactory

First, we create the namespace 'artifactory'
```
[kubi@master ~]$ kubectl create namespace artifactory
namespace/artifactory created
```

Now, let’s start with the secrets:
```
[kubi@master ~]$ kubectl apply -f artifactory-access-config.yaml -f artifactory-binarystore-secret.yaml -f artifactory-secrets.yaml -f artifactory-system.yaml --namespace artifactory
secret/artifactory-access-config created
secret/artifactory-binarystore created
secret/artifactory created
secret/artifactory-systemyaml created

[kubi@master ~]$kubectl get secrets --namespace artifactory
NAME                        TYPE                                  DATA   AGE
artifactory                 Opaque                                2      2m18s
artifactory-access-config   Opaque                                1      4m58s
artifactory-binarystore     Opaque                                1      2m18s
artifactory-systemyaml      Opaque                                1      2m18s
default-token-wcv5h         kubernetes.io/service-account-token   3      6m17s
```
Then, with the Configmaps:
```
[kubi@master ~]$ kubectl apply -f artifactory-migration-scripts.yaml -f artifactory-installer-info.yaml --namespace artifactory
configmap/artifactory-migration-scripts created
configmap/artifactory-installer-info created

[kubi@master ~]$ kubectl get configmaps --namespace artifactory
NAME                            DATA   AGE
artifactory-installer-info      1      50s
artifactory-migration-scripts   3      50s
kube-root-ca.crt                1      10m
```
PersistentVolumeClaims:
```
[kubi@master ~]$ kubectl apply -f artifactory-data-pvc.yaml -f artifactory-backup-pvc.yaml --namespace artifactory
persistentvolumeclaim/artifactory-data-pvc created
persistentvolumeclaim/artifactory-backup-pvc created

[kubi@master ~]$ kubectl get pvc --namespace artifactory
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
artifactory-backup-pvc   Bound    pvc-131d1583-5358-4b6b-afdc-890bb40bf97e   1G         RWX            nfs-storageclass   39s
artifactory-data-pvc     Bound    pvc-36c415c0-62ab-4d30-ab75-aac6cfeee91c   1G         RWX            nfs-storageclass   39s
```
Now it's time for the Statefulset, to create the pod:
```
[kubi@master ~]$ kubectl apply -f artifactory-statefulset.yaml --namespace artifactory
statefulset.apps/artifactory created

[kubi@master ~]$ kubectl get pods --namespace artifactory
NAME            READY   STATUS    RESTARTS   AGE
artifactory-0   1/1     Running   0          2m9s
```
Finally, let's deploy the Service:
```
[kubi@master ~]$ kubectl apply -f artifactory-service.yaml --namespace artifactory
service/artifactory created

[kubi@master ~]$ kubectl get svc --namespace artifactory
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
artifactory   ClusterIP   10.105.10.128   <none>        8082/TCP,8081/TCP   31s
```

## Setup an Ingress Controller
As per [Kubernetes official documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/), an **Ingress** object may provide load balancing, SSL termination and name-based virtual hosting. We require this kind of object to access our JFrog Artifactory instance from outside the cluster.

You must have an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers) to satisfy an Ingress. For this example, we’re using the [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx/blob/master/README.md) officially supported by Kubernetes. We follow the instructions to install it in a [Bare-metal scenario](https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal):
```
[kubi@master ~]$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.1/deploy/static/provider/baremetal/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx configured
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission configured

[kubi@master ~]$ kubectl get services -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.101.240.232   <none>        80:30726/TCP,443:31770/TCP   64s
ingress-nginx-controller-admission   ClusterIP   10.109.184.230   <none>        443/TCP                      64s
```

By default, this installation works in “NopePort” mode. We need to change that to “LoadBalancer”, because we plan to use a separate LoadBalancer that will be responsible to allocate external IPs to our cluster.
```
[kubi@master ~]$ kubectl edit service ingress-nginx-controller -n ingress-nginx
service/ingress-nginx-controller edited

  # We have to look for this entries and modify "type" and save changes
  
  sessionAffinity: None
  type: LoadBalancer # Previous value was NodePort


[kubi@master ~]$ kubectl get services -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.101.240.232   <pending>     80:30726/TCP,443:31770/TCP   3m43s
ingress-nginx-controller-admission   ClusterIP      10.109.184.230   <none>        443/TCP                      3m43s

```
Deploy the Ingress object
```
[kubi@master ~]$ kubectl apply -f artifactory-ingress.yaml --namespace artifactory
ingress.networking.k8s.io/artifactory-ingress created

[kubi@master ~]$ kubectl get ingress --namespace artifactory
NAME                  CLASS   HOSTS   ADDRESS   PORTS   AGE
artifactory-ingress   nginx   *                 80      40s
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

The IP address that appears in the EXTERNAL-IP section of the ingress-nginx-controller is the one through which we can access to our JFrog Artifactory instance.

