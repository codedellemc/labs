# Exploring Kubernetes with ScaleIO Persistent Volumes

### Prerequisites

- An installed Kubernetes cluster
- An installed ScaleIO cluster 
    - can be independent, or converged (Kubernetes minions and ScaleIO co-installed on same hardware)

### Step 1: Establish access to the Kubernetes master node

ssh into a Kubenetes master instance and test operation with these commands.

```
kubectl cluster-info dump
kubectl get nodes
kubectl get pods
kubectl get services
kubectl get deployments
```

#### Step 2: Install REX-Ray on the master node

Rex-Ray is a storage orchestration tool for container schedulers, including Kubernetes. It can be deployed is centralized or distributed architectures, but we will be using a centralized deployment here. It will be deployed inside a Kubernetes pod. 

We will also install an instance of Rex-Ray here on this Kubernetes master node simply to get a command line interface we can use to talk to the central service for some administrative operations.

Continue using the ssh session to the Kubernetes master instance from the previous step. 

```
curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -s -- stable
```

### Step 3: Deploy a REX-Ray controller pod

Rex-Ray is a storage orchestration tool for container schedulers, including Kubernetes. It can be deployed is centralized or distributed architectures, but we will be using a centralized deployment here. It will be deployed inside a Kubernetes pod.

Define a Rex-Ray controller pod and deployment, using a yaml file. This pod will use a preconfigured RexRay controller image hosted on DockerHub. 

```
cat <<EOF >rexraycontroller.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: rexraycontroller
  name: rexraycontroller
spec:
  clusterIP: 10.32.0.11
  type: NodePort
  ports:
    - port: 7979
  selector:
    app: rexraycontroller
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rexraycontroller
  labels:
    app: rexraycontroller
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: rexraycontroller
    spec:
      containers:
      - image: cduchesne/rexray-controller:0.7.0
        imagePullPolicy: Always
        name: rexraycontroller
        env:
        - name: "LIBSTORAGE_HOST"
          value: "tcp://0.0.0.0:7979"
        - name: "LIBSTORAGE_EMBEDDED"
          value: "true"
        - name: "LIBSTORAGE_CLIENT_TYPE"
          value: "controller"
        - name: "LIBSTORAGE_LOGGING_STDOUT"
          value: "TRUE"
        - name: "LIBSTORAGE_LOGGING_HTTPREQUESTS"
          value: "TRUE"
        - name: "LIBSTORAGE_LOGGING_HTTPRESPONSES"
          value: "TRUE"
        - name: "LIBSTORAGE_SERVER_AUTOENDPOINTMODE"
          value: "tcp"
        - name: "LIBSTORAGE_SERVICE"
          value: "scaleio"
        - name: "LIBSTORAGE_SERVER_SERVICES_SCALEIO_DRIVER"
          value: "scaleio"
        - name: "SCALEIO_ENDPOINT"
          value: "https://scaleio-gateway.default.k8s.democluster.com/api"
        - name: "SCALEIO_INSECURE"
          value: "true"
        - name: "SCALEIO_PASSWORD"
          value: "sCaleIO2016"
          # CHANGE THIS PASSWORD!
        - name: "SCALEIO_PROTECTIONDOMAINNAME"
          value: "default"
        - name: "SCALEIO_STORAGEPOOLNAME"
          value: "default"
        - name: "SCALEIO_SYSTEMNAME"
          value: "scaleio"
        - name: "SCALEIO_USERNAME"
          value: "admin"
        ports:
        - name: libstorage-port
          containerPort: 7979
        livenessProbe:
          tcpSocket:
            port: libstorage-port
          # length of time to wait for a pod to initialize
          # after pod startup, before applying health checkin
          initialDelaySeconds: 60
          timeoutSeconds: 3
EOF
```

Use the file to create the pod, and verify.

```
kubectl create -f rexraycontroller.yaml
kubectl describe pod rexraycontroller
```

```
export RRPOD=$(kubectl get pods -l app=rexraycontroller --no-headers | awk '{print $1}')
```

Examine logs to verify Rex-Ray started. 

```
kubectl logs $RRPOD
```

You can test Rex-Ray operation with:

`rexray volume ls -h tcp://10.32.0.11:7979`

### Step 4: Install the FlexRex plugin on minion nodes.

From a root command prompt on each Kubernetes minion node:

```
curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -s — stable
cat <<EOF > /etc/rexray/config.yml
libstorage:
  host: tcp://10.32.0.11:7979
  service: scaleio
EOF
systemctl start rexray
rexray flexrex install
systemctl restart kubelet
```
 
If you have multiple minions, use of the [tmux](https://tmux.github.io/) tool is recommended to do this efficiently. 

### Step 5: Quick and simple, create a PersistentVolume and utilize it in a pod

Continue using the ssh session to the kubernetes master instance from the previous step.

`rexray volume create mysql-1 --size=8 -h tcp://10.32.0.11:7979`

This should create the result shown below. Creation of the specified volume can be confirmed using the ScaleIO GUI console. 

```
admin@ip-172-20-0-9:~$ rexray volume ls -h tcp://10.32.0.11:7979
ID            Name      Status     Size
925e649700000000  mysql-1  available  8
```
 
#### Create a pod containing a stateful application which will use the volume.

Continue using the ssh session to the kubernetes master. In the home directory, create the file pod.yaml with this content. Observe that using a volume mount in this pod definition *decouples the database's data from the container*.:

```
cat <<EOF >pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: mysql
spec:
  containers:
  - image: mysql:5.6
    name: mysql
    env:
      - name: MYSQL_ROOT_PASSWORD
        # CHANGE THIS PASSWORD!
        value: yourpassword
    ports:
      - containerPort: 3306
        name: mysql
    volumeMounts:
      # This name must match the volumes name below.
    - name: mysql-persistent-volume
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-persistent-volume
    flexVolume:
      driver: rexray/flexrex
      fsType: xfs
      options:
        volumeID: mysql-1
        forceAttach: "true"
        forceAttachDelay: "15"
EOF
```

Use the file to create the pod, and verify.

```
kubectl create -f pod.yaml
kubectl describe pod
```

Show the log, which should indicate that MySQL is *ready for connections*:

`kubectl  logs mysql`

### Step 6: Advanced usage, create a PersistentVolumeClaim and utilize it with a Service and Deployment

Users of Kubernetes can request persistent storage for their pods. Administrators can utilize *claims* to allow the storage provisioning and lifecycle to be maintained independently of the applications consuming storage. 

A background process satisfies claims from an inventory of *persistent volumes*. 

Persistent volumes are intended for "network volumes" like GCE persistent disks, and AWS elastic block stores, but can also represent volumes from "overlay" storage providers like ScaleIO. A Persistent Volume (PV) in Kubernetes represents a real piece of underlying storage capacity in the infrastructure. 

A user (application) sees and consumes claims, which hide storage implementation details. This allows an application's configuration to be portable across platforms and providers.

#### Create an ScaleIO volume

`rexray volume create admin-managed-8g-01 --size=8 -h tcp://10.32.0.11:7979`

#### Create a Kubernetes PersistentVolume and PersistentVolumeClaim

Create a Kubernetes persistent volume referencing it:

Create the file pv.yaml with this content:

```
cat <<EOF >pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: small-pv-pool-01
  annotations:
    volume.beta.kubernetes.io/storage-class: "scaleiofast"
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  flexVolume:
    driver: rexray/flexrex
    fsType: xfs
    options:
      volumeID: admin-managed-8g-01
      forceAttach: "true"
      forceAttachDelay: "15"
EOF
```

`kubectl create -f pv.yaml`

Create a Kubernetes persistent volume claim:

Create the file pvc.yaml with this content:

```
cat <<EOF >pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pg-data-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "scaleiofast"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
EOF
```

`kubectl create -f pvc.yaml`

#### Create a Kubernetes Service, and a Deployment that uses the claim

Create the file dep-pvc.yaml with this content:

```
cat <<EOF >dep-pvc.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: postgres-libstorage
  name: postgres-libstorage
spec:
  type: NodePort
  ports:
    - port: 5432
      targetPort: postgres-client
      nodePort: 30032
  selector:
    app: postgres-libstorage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: postgres-libstorage
  labels:
    app: postgres-libstorage
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres-libstorage
    spec:
      containers:
      - image: postgres
        name: postgres
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POSTGRES_PASSWORD
          # CHANGE THIS PASSWORD!
          value: mysecretpassword
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - name: postgres-client
          containerPort: 5432
        volumeMounts:
        - name: pg-data
          mountPath: /var/lib/postgresql/data
      volumes:
        - name: pg-data
          persistentVolumeClaim:
            claimName: pg-data-claim
EOF
```

Now use the file to create the service and deployment.

`kubectl create -f dep-pvc.yaml`

A deployment generates a unique name when it creates a pod. This will lookup the name by label and place it in an environment variable, for convenient reuse later.

```
export PGPOD=$(kubectl get pods -l app=postgres-libstorage --no-headers | awk '{print $1}')
```

Examine logs to verifiy postgres started. Using an interactive session into the pod's container, create a new database named libstoragedemo.

```
kubectl logs $PGPOD
kubectl exec -ti $PGPOD -- bash
su - postgres
psql
CREATE DATABASE flexrexdemo;
\l
\q
exit
exit
```

### Step 7: Engage in Node maintenance to demonstrate migration of a stateful pod to a different cluster host.

This will utilize the PostgreSQL pod created in the previous step.

#### Determine the hosting node

`kubectl get nodes`

should return a list of all nodes like this:

```
NAME            STATUS    AGE
k8s-andromeda   Ready     40d
k8s-hydra       Ready     40d
k8s-phoenix     Ready     40d
k8s-taurus      Ready     40d
```

We can look up the Postgres pod using its label and determine the pod's host node. We will do this and save the node in an environment variable.

Determine which of the hosts is running the Postgres database pod now:

```
export DB_NODE=$(kubectl describe pod -l app=postgres-libstorage | grep Node: | awk -F'[ \t//]+' '{print $2}')
echo $DB_NODE
```

#### Prepare the hosting node for maintenance

We will simulate a planned node outage. (In this example we won't evacuate all pods, only the Postgres pod, but this will be adequate to demonstate stateful application failover.) 

Tell the Kubernetes scheduler to cease directing pods to the node hosting Postgres.

`kubectl cordon $DB_NODE`

Delete the existing pod. The Deployment will automatically bring up a new replacement pod. It will be deployed on a different node because we cordoned off the existing node.

`kubectl delete pod -l app=postgres-libstorage`

Verify that the pod has migrated succesfully.

`kubectl describe pods -l app=postgres-libstorage`

It may take a brief time before the replacement pod shows a "running" status. 

Use an interactive session to verify that the "libstoragedemo" database we created is present:

```
export PGPOD=$(kubectl get pods -l app=postgres-libstorage --no-headers | awk '{print $1}')
kubectl logs $PGPOD
kubectl exec -ti $PGPOD -- bash
su - postgres
psql
\l
\q
exit
exit
```

Finally, tell the scheduler to return the node to service 

`kubectl uncordon $DB_NODE`

### Step 8: Optional: Deploy a StatefulSet 

There are reports of issues with the features described below. Investigation is in progress but at this time please treat this section as preview documentation for a beta stage feature. 

StatefulSet is a Beta feature in Kubernetes 1.5, so this should be considered as preview, and is not recommended for production usage.

The storage for a given StatefulSet pod must be based of a storage class or pre-provisioned by an admin. Since Flex based storage plugins do not support storage class, we will pre-provision storage for this exercise.

Deleting and/or scaling a StatefulSet down will not delete the volumes associated with the StatefulSet. This is done to ensure data safety, which is generally more valuable than an automatic purge of all related StatefulSet resources.

#### Pre-provision volumes backed by ScaleIO

```
rexray volume create redis-24g-1 --size=24 -h tcp://10.32.0.11:7979
rexray volume create redis-24g-2 --size=24 -h tcp://10.32.0.11:7979
rexray volume create redis-24g-3 --size=24 -h tcp://10.32.0.11:7979
```

Create Kubernetes persistent volumes referencing it:

Create the file redis-pv.yaml with this content:

```
cat <<EOF >redis-pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-24g-1
  labels:
    app: redis
spec:
  capacity:
    storage: 24Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: rexray/flexrex
    fsType: xfs
    options:
      volumeID: redis-24g-1
      forceAttach: "true"
      forceAttachDelay: "15"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-24g-2
  labels:
    app: redis
spec:
  capacity:
    storage: 24Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: rexray/flexrex
    fsType: xfs
    options:
      volumeID: redis-24g-2
      forceAttach: "true"
      forceAttachDelay: "15"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-24g-3
  labels:
    app: redis
spec:
  capacity:
    storage: 24Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: rexray/flexrex
    fsType: xfs
    options:
      volumeID: redis-24g-3
      forceAttach: "true"
      forceAttachDelay: "15"
EOF
```

`kubectl create -f redis-pv.yaml`

#### Create a Redis cluster StatefulSet with a Headless Service using a Volume Claim

```
cat <<EOF >redis-ss.yaml
# A headless service to create DNS records
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
  name: redis
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    name: peer
  # *.redis.default.svc.cluster.local
  clusterIP: None
  selector:
    app: redis
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: rd
spec:
  serviceName: "redis"
  replicas: 3
  template:
    metadata:
      labels:
        app: redis
      annotations:
        pod.alpha.kubernetes.io/init-containers: '[
            {
                "name": "install",
                "image": "gcr.io/google_containers/redis-install-3.2.0:e2e",
                "imagePullPolicy": "Always",
                "args": ["--install-into=/opt", "--work-dir=/work-dir"],
                "volumeMounts": [
                    {
                        "name": "opt",
                        "mountPath": "/opt"
                    },
                    {
                        "name": "workdir",
                        "mountPath": "/work-dir"
                    }
                ]
            },
            {
                "name": "bootstrap",
                "image": "debian:jessie",
                "command": ["/work-dir/peer-finder"],
                "args": ["-on-start=\"/work-dir/on-start.sh\"", "-service=redis"],
                "env": [
                  {
                      "name": "POD_NAMESPACE",
                      "valueFrom": {
                          "fieldRef": {
                              "apiVersion": "v1",
                              "fieldPath": "metadata.namespace"
                          }
                      }
                   }
                ],
                "volumeMounts": [
                    {
                        "name": "opt",
                        "mountPath": "/opt"
                    },
                    {
                        "name": "workdir",
                        "mountPath": "/work-dir"
                    }
                ]
            }
        ]'
    spec:
      containers:
      - name: redis
        image: debian:jessie
        ports:
        - containerPort: 6379
          name: peer
        command:
        - /opt/redis/redis-server
        args:
        - /opt/redis/redis.conf
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "/opt/redis/redis-cli -h $(hostname) ping"
          initialDelaySeconds: 15
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /data
        - name: opt
          mountPath: /opt
      volumes:
      - name: opt
        emptyDir: {}
      - name: workdir
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 24Gi
      selector:
        matchLabels:
           app: redis
EOF
```

`kubectl create -f redis-ss.yaml`

#### Test the redis StatefulSet.

```
kubectl get ep
kubectl exec rd-0 -- /opt/redis/redis-cli -h rd-0.redis SET replicated:test true
kubectl exec rd-2 -- /opt/redis/redis-cli -h rd-2.redis GET replicated:test
```

### Step 9: Cleanup 

Delete the MySQL server. Deleting the mount automatically unmounts the volume. Note that by design, volumes until explicitly removed. Volumes can be reused in another pod - even in a different instance of a Kubernetes cluster.  A volume could even be migrated to a different container scheduler platform such as Mesos.

#### Delete MySQL server pod.

`kubectl delete pod mysql`

Delete the Postgres server service and deployment, which will automatically delete the pod.

```
kubectl delete deployment -l app=postgres-libstorage
kubectl delete service postgres-libstorage
```

#### Delete the persistent volume claims. 

```
kubectl delete pvc pg-data-claim
```

#### Delete the persistent volume.

`kubectl delete pv small-pv-pool-01`

#### Delete the ScaleIO volumes.

```
rexray volume rm mysql-1 admin-managed-8g-01 -h tcp://10.32.0.11:7979
```

#### If you performed the optional StatefulSet step

```
kubectl delete statefulset rd
kubectl delete service redis
kubectl delete pvc datadir-rd-0 datadir-rd-1 datadir-rd-2
kubectl delete pv redis-24g-1 redis-24g-2 redis-24g-3
rexray volume rm redis-24g-1 redis-24g-2 redis-24g-3 -h tcp://10.32.0.11:7979
```

## Support

Please file bugs and issues on the GitHub issues page for this project. This is to help keep track and document everything related to this repo. For general discussions and further support you can join the [{code} by Dell EMC Community slack channel](http://community.codedellemc.com/). The code and documentation are released with no warranties or SLAs and are intended to be supported through a community driven process.