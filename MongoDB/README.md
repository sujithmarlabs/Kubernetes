# Mongodb Replica Set on Kubernetes


### Install Minikube


### Install VirtualBox hypervisor
We will install virtualbox 5.* via official reposiories

```
$ sudo apt-get update
$ sudo apt remove virtualbox*
$ wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
$ wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | sudo apt-key add -
$ sudo su
$ echo "deb https://download.virtualbox.org/virtualbox/debian xenial contrib" >> /etc/apt/sources.list
$ apt-get update
$ sudo apt-get install virtualbox virtualbox-ext-pack
```

### Install Kubectl
We will install kubectl from official repositories. Recommened so you will get updates via apt.

```
$ sudo apt-get update && sudo apt-get install -y apt-transport-https
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ sudo touch /etc/apt/sources.list.d/kubernetes.list
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update && sudo apt-get install -y kubectl
$ kubectl version
```

### Install Minikube
Download minukube binary directly from Google repositories.
```
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube
$ sudo mv -v minikube /usr/local/bin
$ minikube version
$ exit
```

### Start Kubernetes Cluster loccally with Minikube
Create and run Kubernetes cluster. This will take some few minutes depending on your Internet connection.
```
$ minikube version
$ minikube start
$ kubectl cluster-info    (Cluster Information)
$ kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
minikube   Ready    master   7m55s   v1.17.3
```

# Mongodb Replica in kubernetes


Run MongoDB Replica Set on Kubernetes using Statefulset and PersistentVolumeClaim. Minikube kubernetes cluster is used for this post.

### Create Secret for Key file

MongoDB will use this key to communicate internal cluster.

```
$ mkdir replica-sets
$ openssl rand -base64 741 > ./replica-sets/key.txt
$ kubectl create secret generic shared-bootstrap-data --from-file=internal-auth-mongodb-keyfile=./replica-sets/key.txt
```
# Deploy MongoDB Replica-Sets YAML

```
cat << EOF > ./replica-sets/mongodb-rc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongod
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      role: mongo
      environment: test
      replicaset: MainRepSet
  template:
    metadata:
      labels:
        role: mongo
        environment: test
        replicaset: MainRepSet
    spec:
      containers:
      - name: mongod-container
        image: mongo:3.4
        command:
        - "numactl"
        - "--interleave=all"
        - "mongod"
        - "--bind_ip"
        - "0.0.0.0"
        - "--replSet"
        - "MainRepSet"
        - "--auth"
        - "--clusterAuthMode"
        - "keyFile"
        - "--keyFile"
        - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
        - "--setParameter"
        - "authenticationMechanisms=SCRAM-SHA-1"
        resources:
          requests:
            cpu: 0.2
            memory: 200Mi
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: secrets-volume
          readOnly: true
          mountPath: /etc/secrets-volume
        - name: mongodb-persistent-storage-claim
          mountPath: /data/db
      volumes:
      - name: secrets-volume
        secret:
          secretName: shared-bootstrap-data
          defaultMode: 256
  volumeClaimTemplates:
  - metadata:
      name: mongodb-persistent-storage-claim
      annotations:
        volume.beta.kubernetes.io/storage-class: "standard"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
EOF    
```
Now Deploy the Yaml

```
$ kubectl create -f ./replica-sets/mongodb-rc.yaml

```

### Wait for Pod running and PVC

```
$ kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
mongod-0   1/1     Running   0          49s
mongod-1   1/1     Running   0          18s
mongod-2   1/1     Running   0          13s

$ kubectl get all
NAME           READY   STATUS    RESTARTS   AGE
pod/mongod-0   1/1     Running   0          5m13s
pod/mongod-1   1/1     Running   0          20m
pod/mongod-2   1/1     Running   0          6m12s

NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/kubernetes        ClusterIP   10.96.0.1    <none>        443/TCP     24m
service/mongodb-service   ClusterIP   None         <none>        27017/TCP   20m

NAME                      READY   AGE
statefulset.apps/mongod   3/3     20m

$ kubectl get pvc
NAME                                        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-persistent-storage-claim-mongod-0   Bound    pvc-e1735bd9-2aaa-4be6-9bec-f06ca16ba970   1Gi        RWO            standard       21m
mongodb-persistent-storage-claim-mongod-1   Bound    pvc-8f9dda85-1d30-4714-ae7c-124977b6512d   1Gi        RWO            standard       21m
mongodb-persistent-storage-claim-mongod-2   Bound    pvc-3596c341-8b91-4539-892d-725e6c6bf8c9   1Gi        RWO            standard       20m

```

### Setup ReplicaSet Configuration

Finally, we need to connect to one of the “mongod” container processes to configure the replica set.

Run the following command to connect to the first container. In the shell initiate the replica set (we can rely on the hostnames always being the same, due to having employed a StatefulSet):

```
$ kubectl exec -it mongod-0 -c mongod-container bash
$ mongo
> rs.initiate({_id: "MainRepSet", version: 1, members: [
       { _id: 0, host : "mongod-0.mongodb-service.default.svc.cluster.local:27017" },
       { _id: 1, host : "mongod-1.mongodb-service.default.svc.cluster.local:27017" },
       { _id: 2, host : "mongod-2.mongodb-service.default.svc.cluster.local:27017" }
 ]});
```

Keep checking the status of the replica set, with the following command, until the replica set is fully initialised and a primary and two secondaries are present:

```
MainRepSet:SECONDARY> rs.status();
```
**output similar to:**
```
{
        "set" : "MainRepSet",
        "date" : ISODate("2020-03-17T14:09:46.530Z"),
        "myState" : 1,
        "term" : NumberLong(1),
        "syncingTo" : "",
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1584454178, 1),
                        "t" : NumberLong(1)
                },
                "appliedOpTime" : {
                        "ts" : Timestamp(1584454178, 1),
                        "t" : NumberLong(1)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1584454178, 1),
                        "t" : NumberLong(1)
                }
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 520,
                        "optime" : {
                                "ts" : Timestamp(1584454178, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2020-03-17T14:09:38Z"),
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "could not find member to sync from",
                        "electionTime" : Timestamp(1584454156, 1),
                        "electionDate" : ISODate("2020-03-17T14:09:16Z"),
                        "configVersion" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongod-1.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 40,
                        "optime" : {
                                "ts" : Timestamp(1584454178, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1584454178, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2020-03-17T14:09:38Z"),
                        "optimeDurableDate" : ISODate("2020-03-17T14:09:38Z"),
                        "lastHeartbeat" : ISODate("2020-03-17T14:09:44.625Z"),
                        "lastHeartbeatRecv" : ISODate("2020-03-17T14:09:45.836Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
                        "syncSourceHost" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                },
                {
                        "_id" : 2,
                        "name" : "mongod-2.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 40,
                        "optime" : {
                                "ts" : Timestamp(1584454178, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1584454178, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2020-03-17T14:09:38Z"),
                        "optimeDurableDate" : ISODate("2020-03-17T14:09:38Z"),
                        "lastHeartbeat" : ISODate("2020-03-17T14:09:44.625Z"),
                        "lastHeartbeatRecv" : ISODate("2020-03-17T14:09:45.738Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
                        "syncSourceHost" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                }
        ],
        "ok" : 1
}

MainRepSet:PRIMARY>
```
**mongodb-0** has become **Primary** and Others two **Secondary Nodes**.

Then run the following command to configure an **“admin”** user (performing this action results in the “localhost exception” being automatically and permanently disabled):

```
MainRepSet:PRIMARY> db.getSiblingDB("admin").createUser({
      user : "main_admin",
      pwd  : "abc123",
      roles: [ { role: "root", db: "admin" } ]
 });

```
**Output is like this**

```
Successfully added user: {
        "user" : "main_admin",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}
```

### Insert Data

Insert Data into mongod-0 pod.

```
> db.getSiblingDB('admin').auth("main_admin", "abc123");
> use test;
> db.testcoll.insert({a:1});
> db.testcoll.insert({b:2});
> db.testcoll.find();

{ "_id" : ObjectId("5e70dee375eb728af03ce40f"), "a" : 1 }
{ "_id" : ObjectId("5e70dee975eb728af03ce410"), "b" : 2 }

> exit
# exit
```

### Verify Cluster Data
**exec** into Secondary Pod (here, mongo-1)

```
$ kubectl exec -it mongod-1 -c mongod-container bash
# mongo
> db.getSiblingDB('admin').auth("main_admin", "abc123");
> db.getMongo().setSlaveOk()
> use test;
> db.testcoll.find();

{ "_id" : ObjectId("5e70dee375eb728af03ce40f"), "a" : 1 }
{ "_id" : ObjectId("5e70dee975eb728af03ce410"), "b" : 2 }

> exit
# exit

```

### Deleting All Resources & Verify PVC

```
$ kubectl delete -f ./replica-sets/mongodb-rc.yaml
$ kubectl get all
$ kubectl get persistentvolumes

```
### Recreate MongoDB

```
$ kubectl apply -f ./replica-sets/mongodb-rc.yaml
$ kubectl get all
```

### Verify Data

As PVC was not deleted, We will still have existing Data.
```
$ kubectl exec -it mongod-0 -c mongod-container bash
# mongo
> db.getSiblingDB('admin').auth("main_admin", "abc123");
> use test;
> rs.slaveOk()
> db.testcoll.find();
> rs.status();
> exit
# exit
```

### Verify Clusterization
Delete **mongod-0 Pod** and keep cheking rs.status(), eventually another node of the remaining two will become Primary Node.

```
$ kubectl delete pods mongod-0
$ kubectl exec -it mongod-1 -c mongod-container bash
# mongo
> db.getSiblingDB('admin').auth("main_admin", "abc123");
> use test;
> rs.slaveOk()
> db.testcoll.find();
> rs.status();
> exit
# exit
```

### Test Connectivity

Test the connectivity by creating a temporary pod in the **default** namespace

```
$ kubectl run -it --rm --restart=Never mongo-cli --image=mongo --command -- /bin/bash
# mongo "mongodb://mongod-0.mongodb-service.default.svc.cluster.local,mongod-1.mongodb-service.default.svc.cluster.local,mongod-2.mongodb-service.default.svc.cluster.local:27017/test"

> db.getSiblingDB('admin').auth("main_admin", "abc123");
> use test;
> rs.slaveOk()
> db.testcoll.find();
```


### Links

(Install Minikube)[https://gist.github.com/gonzaloplaza/f62fdcfdb6aac3d15a0fe0d750715729]
(MongoDB RS)[https://maruftuhin.com/blog/mongodb-replica-set-on-kubernetes/]
(MongoDB RS Another)[http://pauldone.blogspot.com/2017/06/deploying-mongodb-on-kubernetes-gke25.html]
