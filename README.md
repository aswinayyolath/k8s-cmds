![k8s](https://user-images.githubusercontent.com/55191821/126441748-2d1313dd-d7cc-4cd7-88f9-dca522fe59cf.png)

# Kubernetes CLI
## Kubectl
- kubectl is almost the only tool we'll need to talk to kubernetes
- It is a rich CLI tool around the Kubernetes API
- On our machines, there is a `~/.kube/config` file with:
  - The kubernetes API address
  - The path to our TLS certificates used to authenticate
- We can also use the --kubeconfig flag to pass a config file

### List nodes
- kubectl get nodes

![kubectl-get-node](https://user-images.githubusercontent.com/55191821/126443144-4647f947-f5e4-4b34-a330-6f1f6a43874e.png)
- To get more about nodes
  - kubectl get nodes -o wide
![wide-output](https://user-images.githubusercontent.com/55191821/126443573-a61c312b-3385-4ef4-ade7-462a08d58b12.png)

- To get nodes details in yaml
  - kubectl get nodes -o yaml

- Show the capacity of all our nodes as a stream of JSON objects
   - kubectl get nodes -o json | jq ".items[] | {name:.metadata.name} + .status.capacity"
![jq-kubectl](https://user-images.githubusercontent.com/55191821/126445327-f488d236-d7e5-4c57-9dca-8bce80e62939.png)

- `kubectl describe` needs a resource type and (optionally) a resource name
- It is possible to provide a resource name prefix
- Syntax : `kubectl describe node/<node>` or `kubectl describe node <node>` Example: `kubectl describe node/minikube`

## Exploring types and definitions
- We can list all available resource type by running `kubectl api-resources`
- We can view the definition of resource type with `kubectl explain <type>` example: `kubectl explain node.spec`
- To get the list of all fields and sub-fields use `kubectl explain node --recursive` (Example)

## Services
- A service is a stable endpoint to connect to something
- `kubectl get services` or `kubectl get svc`

## Namespaces
- Namespaces allows us to segregate resources
- Namespaces will filter the view by default
- Get namespaces using `kubectl get namespaces`
- In K8S we can organize resources in namespaces
- We can think of a namespace as a virtual cluster inside a Kubernetes cluster

    ```
    $ kubectl get namespaces
    NAME              STATUS   AGE
    default           Active   24m
    kube-node-lease   Active   24m
    kube-public       Active   24m
    kube-system       Active   24m
    $ 
    ```
- `kubectl get pods --all-namespaces`
- To get the pods of the default namespace use `kubectl get pods -n default`
- To create a new namespace use `kubectl create namespece <namespace_name>` Example: `kubectl create namespace my-namespace`
    ```
    $ kubectl create namespace my-namespace
    namespace/my-namespace created
    $ kubectl get namespaces
    NAME              STATUS   AGE
    default           Active   4m1s
    kube-node-lease   Active   4m2s
    kube-public       Active   4m2s
    kube-system       Active   4m2s
    my-namespace      Active   76s
    $
    ```
- We can also create namespace using namespace configuration file

### What is the use of namespaces?
- **Structure** your components
- **Avoid confilts** between teams
- **Share services** between different environments
- **Access and Resource Limits** on Namespace levels

**Note** : *We can't access most of the resource from another namespace for example if we have a configmap in project A namespace that reference a Database service we can use that configmap in project B namespace. Same applies to Secrets. However services can be shared across namespaces. Some of the components lives globally in a cluster and we can't isolate them Example: Volume,Node*
- `kubectl api-resources --namespaced=false` lists components that are not bound to a namespace
- `kubectl api-resources --namespaced=true`  lists components that are bound to a namespace

### How to create components in Namespace
- By default if we don't provide namespace to a component it creates them in a default namespace
- We can create a resource inside a namespace by specifing the name of the namespace in the --namespace flag as below
    ```
    $ touch sample-pod.yaml
    $ vim sample-pod.yaml 
    $ kubectl apply -f sample-pod.yaml --namespace my-namespace 
    pod/static-web created
    $ kubectl get pods --namespace my-namespace
    NAME         READY   STATUS    RESTARTS   AGE
    static-web   1/1     Running   0          30s
    $ 
    ```
- We can also add the namespace inside the yaml configuration. In that case we need to specify namespace using the namespace flag
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: static-web
      namespace: my-namespace
      labels:
        role: myrole
    spec:
      containers:
        - name: web
          image: nginx
          ports:
            - name: web
              containerPort: 80
              protocol: TCP
    ```
    
    ```
    $ kubectl delete pods static-web  --namespace my-namespace
    pod "static-web" deleted
    $ kubectl apply -f sample-pod.yaml
    pod/static-web created
    $ kubectl get pods --namespace my-namespace 
    NAME         READY   STATUS    RESTARTS   AGE
    static-web   1/1     Running   0          7m32s
    $ 
    ```
**Note:** *kubens* allows us to easily switch between Kubernetes namespaces. Usage `kubens <namespace>`

## Deployment
- A deployment is a high-level construct
  - Allows scaling, rolling updates, rollbacks
  - multiple deployments can be used
  - delegates pods management to replica sets

- A replica set is a low-level construct
  - Makes sure that a given number of identical pods are running
  - Allows scaling
  - Rarely used directly

- To create a deployment using yaml use `kubectl apply -f <deployment-name.yaml>`
    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      labels:
        app: nginx
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.14.2
            ports:
            - containerPort: 80
    ```
    
    ```
    $ touch nginx-deployment.yaml
    $ vim nginx-deployment.yaml 
    $ kubectl apply -f nginx-deployment.yaml 
    deployment.apps/nginx-deployment created
    $ kubectl get deployment
    NAME               READY   UP-TO-DATE   AVAILABLE   AGE
    nginx-deployment   1/3     3            1           23s
    $ 
    ```
- Delete one pod and test self healing in action
    ```
    $ kubectl get all
    NAME                                    READY   STATUS    RESTARTS   AGE
    pod/nginx-deployment-574b87c764-27sxx   1/1     Running   0          4m56s
    pod/nginx-deployment-574b87c764-fhrtb   1/1     Running   0          4m56s
    pod/nginx-deployment-574b87c764-xtgsj   1/1     Running   0          4m56s

    NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   8m29s

    NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/nginx-deployment   3/3     3            3           4m56s

    NAME                                          DESIRED   CURRENT   READY   AGE
    replicaset.apps/nginx-deployment-574b87c764   3         3         3       4m56s
    $ kubectl delete  pod/nginx-deployment-574b87c764-27sxx
    pod "nginx-deployment-574b87c764-27sxx" deleted
    $ 
    ```
- Use `watch kubectl get pods` to see how a new pod is added to the cluster

![re-creating](https://user-images.githubusercontent.com/55191821/126512732-460ae0ef-83b8-4c6c-834f-22ac5614e77c.png)

- To get all the details of deployment including status use `kubectl get deployment <deployment-name> -o yaml` 
  Example: `kubectl get deployment nginx-deployment -o yaml`
- We can also create deploymet using commands like `kubectl create deployment pingpong --image=alpine --replicas=3`
- To update a deployment use (here scale up replicas) `kubectl scale deployment pingpong --replicas=3`

## Services
### Exposing  containers
- `kubectl expose` creates a *service* for existing pods  
- A service is a stable address for a pod (or a bunch of pods)
- If we want to connect to our pod(s), we need to create a service
- Once a service is created, CoreDNS will allow us to resolve it by name

### Basic service types
- Cluster IP(default type)
  - A virtual IP address is allocated for the service (in an internal, private range)
  - This IP is reachable only from within the cluster (nodes and pods)
  - Our code can connect to the service using original port number
- NodePort 
  - A port is allocated for the service (by default, in range 30000-32768)
  - That port is made available on all our nodes and anybody can connect to it
  - Our code must be changed to connect to that new port number
  
**Note** : Cluster IP and NodePort are always available. Under the hood : `kube-proxy` is using a userland proxy and a bunch of `iptables` rules

- LoadBalancer
  - An external load balancer is allocated for the service
  - Available only when underlaying infrastructure provides some load balancer as service (e.g. AWS, Azure, GCE)
- ExternalName
  - The DNS entry managed by coreDNS will just be a `CNAME` to a provided record
  - No ports, no IP address, no nothing else is allocated 

## K8S network model
- Everything can reach everything
- No address translation
- No port translation
- No new protocol
- Every Pod gets its own IP address
- pods on a node can communicate with all pods on all nodes without NAT
- agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node
