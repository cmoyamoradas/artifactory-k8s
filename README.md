# Deploy JFrog Artifactory on a Kubernetes cluster

These are some Kubernetes manifests to deploy a JFrog Artifactory on a Kubernetes cluster.

These manifests are configured to use an external PostgreSQL database and use NFS mounts as storage system for the shared filesystem.

## On this page
- Pre-requisites
- Setup a NFS Server
- Setup NFS Clients
- Dynamic NFS provisioning
- Deploying a JFrog Artifactory instance
- Setup an Ingress Controller
- Deploy an Ingress object
- Setup a load balancer

## Pre-requisites
- You need a Kubernetes cluster up-and-running in order to run this example.
- You need a shared storage. For this example, we will use an external server (external to the Kubernetes cluster, I mean) and NFS protocol. In the following sections we’re going to describe how to configure this.
- You need to setup a load balancer to provision an external IP to make accessible the application from outside of the Kubernetes cluster.

## Setup a NFS Server
I will consider that we’re setting up the NFS Server on CentOS 7, to stay consistent with the other servers.

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
[kubi@master ~]$ kubectl create namespace nfs
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
    namespace: nfs
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
    namespace: nfs
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

And finally, we can proceed with the deploy of the nfs provisioner, updating the nfs server IP address and the path to the nfs folder:

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
To setup a JFrog Artifactory instance in Kubernetes, we will require to deploy several objects and always respecting an order.
First, we create a dedicated namespace:
```
[kubi@master ~]$ kubectl create namespace artifactory
namespace/artifactory created
```

Then, we generate the master and join keys, and create a secret in Kubernetes. Important to name the secret "artifactory"
```
[kubi@master ~]$ export MASTER_KEY=$(openssl rand -hex 32)
echo ${MASTER_KEY}
93816047a3c0ea7c77f904db87579e2530dc3749ebe0dd113b2413f68af18cc5

[kubi@master ~]$ export JOIN_KEY=$(openssl rand -hex 32)
echo ${JOIN_KEY}
d8ce3ef3af5f769a9e0e92d2c62ea27d4752d1c7d790f821f0864f48f9dd3308

[kubi@master ~]$ kubectl create secret generic artifactory --from-literal=master-key=${MASTER_KEY} --from-literal=join-key=${JOIN_KEY} --namespace artifactory
secret/artifactory created
```

Now, we create a new secret for the database connection information:
```
[kubi@master ~]$ kubectl create secret generic artifactory-database-creds --from-literal=db-user='[USERNAME]' --from-literal=db-password='[PASSWORD]' --from-literal=db-url='jdbc:postgresql://[HOSTANME|IP]:[PORT]/[DATABASE NAME]?sslmode=disable' --namespace artifactory
secret/artifactory-database-creds created
```

We need to apply the following three manifests to create additional secrets that we'll reuse in other manifests:
```
[kubi@master ~]$ kubectl apply -f artifactory-access-config.yaml -f artifactory-binarystore-secret.yaml  -f artifactory-system.yaml  --namespace artifactory
secret/artifactory-access-config created
secret/artifactory-binarystore created
secret/artifactory-systemyaml created
```
Let's check the secrets before moving forward:
```
[kubi@master ~]$ kubectl get secrets --namespace artifactory
NAME                         TYPE                                  DATA   AGE
artifactory                  Opaque                                2      7m57s
artifactory-access-config    Opaque                                1      105s
artifactory-binarystore      Opaque                                1      105s
artifactory-database-creds   Opaque                                3      6m29s
artifactory-systemyaml       Opaque                                1      105s
default-token-xr5dc          kubernetes.io/service-account-token   3      15m
```

Let's continue with the Configmaps:
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

It's the time to create the PersistentVolumes. But before applying the manifest, we need to create the paths that we intend to configure on them.
That is, we go to the NFS Server, and under the root NFS directory (e.g. /var/nfsshare), we create a folder "data" and another "backup", both required in an HA Artifactory installation.
These two paths are the ones that we need to configure in the following manifests:
Important NOTE: Don't forget to make "artifactoy" user (uid 1030) the owner of these two folders.
```
[kubi@master ~]$ kubectl apply -f artifactory-data-pv.yaml -f artifactory-backup-pv.yaml --namespace artifactory
persistentvolume/artifactory-data-pv created
persistentvolume/artifactory-backup-pv created

[kubi@master ~]$ kubectl get pv --namespace artifactory
NAME                    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                STORAGECLASS       REASON   AGE
artifactory-backup-pv   4Gi        RWX            Retain           Available   artifactory/artifactory-backup-pvc   nfs-storageclass            37s
artifactory-data-pv     4Gi        RWX            Retain           Available   artifactory/artifactory-data-pvc     nfs-storageclass            37s
```

Now, let's apply the PersistentVolumeClaim manifests:
```
[kubi@master ~]$ kubectl apply -f artifactory-data-pvc.yaml -f artifactory-backup-pvc.yaml --namespace artifactory
persistentvolumeclaim/artifactory-data-pvc created
persistentvolumeclaim/artifactory-backup-pvc created

[kubi@master ~]$ kubectl get pvc --namespace artifactory
NAME                     STATUS   VOLUME                  CAPACITY   ACCESS MODES   STORAGECLASS   AGE
artifactory-backup-pvc   Bound    artifactory-backup-pv   4Gi        RWX                           38s
artifactory-data-pvc     Bound    artifactory-data-pv     4Gi        RWX                           38s
```

At this stage, we can already apply the statefulset manifest, to create the first pod:
```
[kubi@master ~]$ kubectl apply -f artifactory-statefulset.yaml --namespace artifactory
statefulset.apps/artifactory created

[kubi@master ~]$ kubectl get pods --namespace artifactory
NAME            READY   STATUS    RESTARTS   AGE
artifactory-0   1/1     Running   0          2m9s
```
Notice that it can take a few minutes to have the pod in a "ready" status.

Finally, let's deploy the service:
```
[kubi@master ~]$ kubectl apply -f artifactory-service.yaml --namespace artifactory
service/artifactory created

[kubi@master ~]$ kubectl get svc --namespace artifactory
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
artifactory   ClusterIP   10.108.102.0   <none>        8082/TCP,8081/TCP   12s
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
## Setup a load balancer
The load balancer is the element in the architecture that will provide the external IP to the ingress service.
We can face two scenarios:
- Kubernetes cluster deployed on a bare metal or virtualized infrastructure, either on premise or Cloud IaaS provider (AWS, GCP, Azure, etc.)
- Kubernetes cluster deployed on a Cloud Kubernetes service such as GKE, AKE, EKS, etc.

I'm just covering here the first scenario, split in two sub-scenarios

### Bare metal or virtualized on premise infrastructure
For this scenario, we can rely on [Metallb](https://metallb.universe.tf). To install Metallb on Kubernetes cluster described in the [MetalLB official documentation](https://metallb.universe.tf/installation/)

After having followed the installation process, if we have a look at the ingress services, we should already see that an external IP has been assigned to the ingress service:
```
[kubi@master ~]$ kubectl get services -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.101.240.232   10.186.0.21   80:30726/TCP,443:31770/TCP   4h33m
ingress-nginx-controller-admission   ClusterIP      10.109.184.230   <none>        443/TCP                      4h33m
```

The IP address that appears in the EXTERNAL-IP section of the ingress-nginx-controller service is the one through which we can access to our JFrog Artifactory instance.

### VMs on Cloud IaaS provider (AWS, GCP, Azure)
Unfortunately, in a Cloud provider scenario, where the kubernetes cluster is deployed on some VMs, the majority of providers don't support the network protocols that Metallb requires.
More info [here](https://metallb.universe.tf/installation/clouds/).

The alternative would be to configure a load balancer in your Cloud provider with a static IP address and then configure the ingress service with this particular IP address.
```
[kubi@master ~]$ kubectl edit svc ingress-nginx-controller --namespace ingress-nginx
```
```
...
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.101.240.232
  clusterIPs:
  - 10.101.240.232
  externalIPs: ¢ New property to be added
  - 10.186.0.21 ¢ Static IP address from the load balancer
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  loadBalancerIP: 10.186.0.21 # New property to be added, with the static IP address from the load balancer
 ...
```
## Configure and scale your Artifactory deployment
At this point, you should be able to access to Artifactory through your preferred browser.
It's time to configure the licenses. Add to valid licenses before scaling the deployment.
Once the licenses are already uploaded, proceed with the scaling as follows:
```
[kubi@master ~]$ kubectl scale statefulset artifactory --replicas=2 --namespace artifactory
statefulset.apps/artifactory scaled

[kubi@master ~]$ kubectl get pods --namespace artifactory
NAME            READY   STATUS    RESTARTS   AGE
artifactory-0   1/1     Running   0          13m
artifactory-1   1/1     Running   0          2m47s
```
Notice that it can take a few minutes to get the second pod on "ready" status.