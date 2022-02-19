1. Overview
When working with Kubernetes, we lack a tool that helps in local development — a tool that can run local Kubernetes clusters using Docker containers as nodes.

In this tutorial, we'll explore Kubernetes with kind. Primarily a testing tool for Kubernetes, kind is also handy for local development and CI.

2. Setup
As a prerequisite, we should make sure Docker is installed in our system. An easy way to install Docker is using the Docker Desktop appropriate for our operating system (and processor, in the case of macOS).

2.1. Install Kubernetes Command-Line
First, let's install the Kubernetes command-line, kubectl.On macOS, we can install it using Homebrew:

$ brew install kubectl
We can verify the successful installation by using the command:

2.2. Install kind
Next, we'll install kind using Homebrew on macOS:

$ brew install kind
To verify the successful installation, we can try the command:

$ kind version
kind v0.11.1 go1.15.6 darwin/amd64
However, if the kind version command doesn't work, please add its location to the PATH variable.

Similarly, for the Windows operating system, we can download kind using curl:

curl -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.11.1/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\kind\kind.exe
3. Kubernetes Cluster
Now, we're all set to use kind to prepare the local development environment for Kubernetes.

3.1. Create Cluster
First, let's create a local Kubernetes cluster with the default configuration:

$ kind create cluster
By default, a cluster named kind will be created. However, we can provide a name to the cluster using the –name parameter:

$ kind create cluster --name baeldung-kind
Creating cluster "baeldung-kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-baeldung-kind"
You can now use your cluster with:
kubectl cluster-info --context kind-baeldung-kind
Thanks for using kind! 😊
Also, we can use a YAML config file to configure the cluster. For instance, let's write a simplistic configuration in the baeldungConfig.yaml file:

kind: Cluster
apiVersion: kind.x-k8s.io/v1
name: baeldung-kind
Then, let's create the cluster using the configuration file:

$ kind create cluster --config baeldungConfig.yaml
Additionally, we can also provide a specific version of the Kubernetes image while creating a cluster:

$ kind create cluster --image kindest/node:v1.20.7

3.2. Get Cluster
Let's check the created cluster by using the get command:

$ kind get clusters
baeldung-kind
Also, we can confirm the corresponding docker container:

$ docker ps
CONTAINER ID  IMAGE                 COMMAND                 CREATED    STATUS        PORTS                      NAMES
612a98989e99  kindest/node:v1.21.1  "/usr/local/bin/entr…"  1 min ago  Up 2 minutes  127.0.0.1:59489->6443/tcp  baeldung-kind-control-plane
Or, we can confirm the nodes via kubectl:

$ kubectl get nodes
NAME                          STATUS   ROLES                  AGE   VERSION
baeldung-kind-control-plane   Ready    control-plane,master   41s   v1.21.1
3.3. Cluster Details
Once a cluster is ready, we can check the details using the cluster-info command on kubectl:

$ kubectl cluster-info --context kind-baeldung-kind
Kubernetes master is running at https://127.0.0.1:59489
CoreDNS is running at https://127.0.0.1:59489/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Also, we can use the dump parameter along with the cluster-info command to extract the detailed information about a cluster:

$ kubectl cluster-info dump --context kind-baeldung-kind

3.4. Delete Cluster
Similar to the get command, we can use the delete command to remove a specific cluster:

$ kind delete cluster --name baeldung-kind

4. Ingress Controller
4.1. Configure
We'll require an ingress controller to establish a connection between our local environment and the Kubernetes cluster.

Therefore, we can use kind‘s config options like extraPortMappings and node-labels:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: baeldung-kind
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"    
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
Here, we've updated our baeldungConfig.yaml file to set the configurations for the ingress controller, mapping the container port to the host port. Also, we enabled the node for the ingress by defining ingress-ready=true.

Then, we must recreate our cluster with the modified configuration:

kind create cluster --config baeldungConfig.yaml

4.2. Deploy
Then, we'll deploy the Kubernetes supported ingress NGINX controller to work as a reverse proxy and load balancer:

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
Additionally, we can also use AWS and GCE load balancer controllers.

5. Deploying a Service Locally
Finally, we're all set to deploy our service. For this tutorial, we can use a simple http-echo web server available as a docker image.

5.1. Configure
So, let's create a configuration file that defines the service and use ingress to host it locally:

kind: Pod
apiVersion: v1
metadata:
  name: baeldung-app
  labels:
    app: baeldung-app
spec:
  containers:
  - name: baeldung-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=Hello World! This is a Baeldung Kubernetes with kind App"
---
kind: Service
apiVersion: v1
metadata:
  name: baeldung-service
spec:
  selector:
    app: baeldung-app
  ports:
  # Default port used by the image
  - port: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: baeldung-ingress
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/baeldung"
        backend:
          service:
            name: baeldung-service
            port:
              number: 5678
---
Here, we've created a pod named baeldung-app with the text argument and a service called baeldung-service.

Then, we set up ingress networking to the baeldung-service on the 5678 port and through the /baeldung URI.

5.2. Deploy
Now that we're ready with all the configuration and our cluster integrates with the ingress NGINX controller, let's deploy our service:

$ kubectl apply -f baeldung-service.yaml
We can check the status of the services on kubectl:

$ kubectl get services
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
baeldung-service   ClusterIP   10.96.172.116   <none>        5678/TCP   5m38s
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP    7m5s
That's it! Our service is deployed and should be available at localhost/baeldung:

$ curl localhost/baeldung
Hello World! This is a Baeldung Kubernetes with kind App
Note: if we encounter any error related to the validate.nginx.ingress.kubernetes.io webhook, we should delete the ValidationWebhookConfiguration:

$ kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
validatingwebhookconfiguration.admissionregistration.k8s.io "ingress-nginx-admission" deleted
Then, deploy the service again.

PS:
https://ichi.pro/pl/kubernetes-nodeport-czy-loadbalancer-czy-ingress-kiedy-powinienem-uzywac-czego-160730578438624
https://www.baeldung.com/ops/kubernetes-kind