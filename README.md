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

$ minikube addons enable dashboard

$ kubectl apply -f ./yamlfile/dashboard/kubernetes-dashboard.yaml

$ kubectl get pods -n kubernetes-dashboard -w
