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
- We can also add the namespace inside the yaml configuration. In that case we don't need to specify namespace using the namespace flag
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
