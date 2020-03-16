# Kubernetes

$ minikube version

$ kubectl cluster-info    (Cluster Information)

$ kubectl get nodes
```
NAME       STATUS   ROLES    AGE     VERSION
minikube   Ready    master   7m55s   v1.17.3
```
$ kubectl create deployment first-deployment --image=katacoda/docker-http-server

$ kubectl get pods

$ kubectl expose deployment first-deployment --port=80 --type=NodePort

$ export PORT=$(kubectl get svc first-deployment -o go-template='{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}')

$ echo "Accessing host01:$PORT"

$ curl host01:$PORT

### Enable Dashboard

```
$ minikube addons enable dashboard

$ kubectl apply -f ./yamlfile/dashboard/kubernetes-dashboard.yaml

$ kubectl get pods -n kubernetes-dashboard -w
```

### Kubernetes Task

The following command will launch a deployment called http which will start a container based on the Docker Image katacoda/docker-http-server:latest.
```
$ kubectl run http --image=katacoda/docker-http-server:latest --replicas=1
```
```
$ kubectl get deployments
```
To find out what Kubernetes created you can describe the deployment process.

```
$ kubectl describe deployment http
```

With the deployment created, we can use kubectl to create a service which exposes the Pods on a particular port.

Expose the newly deployed http deployment via kubectl expose. The command allows you to define the different parameters of the service and how to expose the deployment.

**Use the following command to expose the container port 80 on the host 8000 binding to the external-ip of the host.**

```
$ kubectl expose deployment http --external-ip="172.17.0.60" --port=8000 --target-port=80
```


```
$ curl http://172.17.0.60:8000
<h1>This request was processed by host: http-774bb756bb-mftq5</h1>

```


### Kubectl Run and Expose

With kubectl run it's possible to create the deployment and expose it as a single command.

**Use the command command to create a second http service exposed on port 8001.**
```
$ kubectl run httpexposed --image=katacoda/docker-http-server:latest --replicas=1 --port=80 --hostport=8001

$ curl http://172.17.0.60:8001
```

**Under the covers, this exposes the Pod via Docker Port Mapping. As a result, you will not see the service listed using**

```
$ kubectl get svc

$ docker ps | grep httpexposed
```

### Scale Containers

Scaling the deployment will request Kubernetes to launch additional Pods. These Pods will then automatically be load balanced using the exposed Service.
```
$ kubectl scale --replicas=3 deployment http
```

Listing all the pods, you should see three running for the http deployment

```
$ kubectl get pods
```
Once each Pod starts it will be added to the load balancer service. By describing the service you can view the endpoint and the associated Pods which are included.

```
$ kubectl describe svc http

$ curl http://172.17.0.60:8000
```


### Deploy Containers Using YAML

One of the most common Kubernetes object is the deployment object. The deployment object defines the container spec required, along with the name and labels used by other parts of Kubernetes to discover and connect to the application.

Copy the following definition to the editor. The definition defines how to launch an application called webapp1 using the Docker Image katacoda/docker-http-server that runs on Port 80.

```
$ kubectl create -f ./yamlfile/sampledeployment/deployment.yaml

$ kubectl get deployment

$ kubectl describe deployment webapp1
```

Kubernetes has powerful networking capabilities that control how applications communicate. These networking configurations can also be controlled via YAML.

Copy the Service definition to the editor. The Service selects all applications with the label webapp1. As multiple replicas, or instances, are deployed, they will be automatically load balanced based on this common label. The Service makes the application available via a NodePort.

```
$ kubectl create -f service.yaml

$ kubectl get svc

$ kubectl describe svc webapp1-svc

$ curl host01:30080
```
