# Vinrouter Demo with NGINX Ingress

Search and replace myregistry.com with the uri of your private container registry

```bash
myregistry.com => 664341837355.dkr.ecr.us-east-2.amazonaws.com
``` 

## Amazon Elastic Kubernetes Service (EKS) Setup

We will use [AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) for this demo. Feel free to use what ever Kubernetes platform you like. The following section has notes on setting up EKS and AWS 

### Setup Amazon EKS

1. We will need AWS Command Line Interface (CLI) installed on your client machine to manage your AWS services. See [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

2. See [Getting started with Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)

3. To create or update the kubeconfig file for your cluster, run the following command:

```bash
# aws eks --region region update-kubeconfig --name cluster_name

aws eks --region us-east-2 update-kubeconfig --name armand-eks
```

#### Setup AWS Container Registry (ECR)

**Tip:** For additional registry authentication methods, including the Amazon ECR credential helper, see [Registry Authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html#registry_auth) 

Take note of the image URIs in your private registry that is created from the following steps. We will need it for when we push container images to the registry

1. Retrieve an authentication token and authenticate your Docker client to your registry **using the AWS CLI:**


```bash
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin myregistry.com
```

2. Create an ECR repository for `nginx-ingress`

```bash
aws ecr create-repository --repository-name nginx-ingress --region us-east-2
```

Example output:

```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-2:xxxxxxxxx:repository/nginx-ingress",
        "registryId": "xxxxxxxxxxx",
        "repositoryName": "nginx-ingress",
        "repositoryUri": "myregistry.com/nginx-ingress",
        "createdAt": "2020-11-07T23:26:04-07:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
```

2. Create an ECR repository for `dev-backend`

```bash
aws ecr create-repository --repository-name dev-backend --region us-east-2
```

3. Create an ECR repository for `prod-backend`

```bash
aws ecr create-repository --repository-name prod-backend --region us-east-2
```

4. Create an ECR repository for `vinrouter`

```bash
aws ecr create-repository --repository-name vinrouter --region us-east-2
```

## Deploy Demo Environment

In this section we need to setup the dummy backends: "dev" (`dev-backend`) and "prod" `prod-backend`, and the "vinrouter" `vinrouter`

### Build and push "Dev backend"

From the **project root** find the `apps` folder where the docker file, NGINX config and Makefile reside.

Have ready your of container image URIs in your private registry. We will need to supply this URI as the `IMAGE` argument value

We Will deploy these services in the `default` namespace to demonstrate the Ingress routing to different namespaces

1. Build `dev-backend` container, push to registry and test:

```bash
cd apps/dev-backend

# make build IMAGE=[image_url]/[image_name]
# e.g.
make build IMAGE=myregistry.com/dev-backend
make push IMAGE=myregistry.com/dev-backend
make test IMAGE=myregistry.com/dev-backend

```

### Build and push "prod backend"

1. From the project root, build `prod-backend` container, push to registry and test:

```bash
cd apps/prod-backend

# make build IMAGE=[image_url]/[image_name]
# e.g.
make build IMAGE=myregistry.com/prod-backend
make push IMAGE=myregistry.com/prod-backend
make test IMAGE=myregistry.com/prod-backend

```

### Build and push "vinrouter"

1. From the project root, build `vinrouter` container, push to registry and test:

```bash
cd apps/vinrouter

# make build IMAGE=[image_url]/[image_name]
# e.g.
make build IMAGE=myregistry.com/vinrouter
make push IMAGE=myregistry.com/vinrouter
make test IMAGE=myregistry.com/vinrouter

```

### Deploy the Demo services to the Kubernetes cluster: `dev-backend`, `prod-backend` and `vinrouter`

1. From the project root, find the manifests (`.yaml` files) in the `k8s` folder:
```bash
cd k8s
```

2. `dev-backend` and `prod-backend` service and deployments are defined in the `backends.yaml` file. Before deploying we need to insert the correct uri to your container images in the manifest:

For Example:

```yaml
# backends.yaml

      containers:
      - name: dev-backend
        image: myregistry.com/dev-backend:latest
        ports:
        - containerPort: 80
# ... 
      containers:
      - name: prod-backend
        image: myregistry.com/prod-backend:latest
        ports:
        - containerPort: 80
```

3. We will do the same for `vinrouter`. Modify `vinrouter.yaml` and include the correct uri to your container images:

```bash
# vinrouter.yaml

      containers:
      - name: prod-backend
        image: myregistry.com/vinrouter
        ports:
        - containerPort: 80
```

3. Now we deploy the backends, `dev-backend` and `prod-backend`, and `vinrouter` into your cluster in the `default` namespace (no `namespace` is provided in the `kubectl` command):

```bash
kubectl apply -f backends.yaml

deployment.apps/dev-backend created
service/dev-backend-svc created
deployment.apps/prod-backend created
service/prod-backend-svc created
```

4. And now Deploy the `vinrouter` into your cluster in the `default` namespace (no `namespace` is provided in the `kubectl` command):


```bash
kubectl apply -f vinrouter.yaml

deployment.apps/vinrouter created
service/vinrouter-svc created
```

5. The Pods should be ready in a moment:

```bash
# Run `get pods` on the default namespace
kubectl get pods

NAME                            READY   STATUS    RESTARTS   AGE
dev-backend-65d795b777-gx8b2    1/1     Running   0          4m59s
prod-backend-848bc9c989-5qpkx   1/1     Running   0          4m59s
vinrouter-64878bfcb4-bp9lp      1/1     Running   0          16s

```

## Prepare for NGINX Ingress helm installation

For this demo we will be using [helm package manager](https://helm.sh/) for Kubernetes. 

### Build and push NGINX Plus Docker image

For our custom build (with NJS) will be making tweaks on the [NGINX Plus helm install guide](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)

1. Getting the Chart Sources by git cloning the **NGINX Plus Ingress controller** repo:

```bash
git clone https://github.com/nginxinc/kubernetes-ingress
```

2. For NGINX Plus Ingress we need to make sure the NGINX Plus certificate (`nginx-repo.crt`) and the key (`nginx-repo.key`) located in the project:

```bash
cd kubernetes-ingress
ls nginx-repo.*
nginx-repo.crt  nginx-repo.key
```

2. Copy the provided and modified Dockerfile with NJS module 

```bash
cp ../nginx-plus-ingress/DockerfileForPlus kubernetes-ingress/build
```

3. Using the Makefile provided in the repo, build the NGINX ingress image and push it to our private container registry with the tag `1.9.0-njs` using the makefile argument `VERSION=1.9.0-njs`, this signifies our NJS build

```bash
# Where `myregistry.example.com/nginx-ingress` defines the container image uri in your private registry where the image will be pushed. Substitute that value with the repo in your private registry.
# e.g.
#make DOCKERFILE=DockerfileForPlus PREFIX=myregistry.example.com/nginx-plus-ingress

make DOCKERFILE=DockerfileForPlus PREFIX=myregistry.com/nginx-ingress VERSION=1.9.0-njs
```

### Create `nginx-ingress` namespace

We will deploy NGINX ingress in its own namespace

1. Create a designated namespace, i.e. `nginx-ingress`, for our NGINX Ingress:

```bash
kubectl create namespace nginx-ingress

namespace/nginx-ingress created

kubectl get namespaces

NAME              STATUS   AGE
default           Active   8h
kube-node-lease   Active   8h
kube-public       Active   8h
kube-system       Active   8h
nginx-ingress     Active   2s # <-- there it is!

```

### Deploy the NJS Config Map (`njs-configmap.yaml`) referenced in the helm values (`values.yaml`) file

1. From the project root, find the configMap, `njs-configmap.yaml` in the `k8s` folder

2. Apply the `njs-configmap.yaml` `configMap`

```bash
kubectl apply -f k8s/njs-configmap.yaml
```

### Deploy the `virtual-server` configuration

**Important:** the `virtualServer` must be deployed in the same `namespace` as the `service` and `deployement` (`dev-backend`, `prod-backend` and `vinrouter`) the ingress is routing to

1. From the project root, find the manifest, `virtual-server.yaml` in the `k8s` folder:

2. Deploy the `virtual-server.yaml` configMap in the `default` namespace

```bash
kubectl apply -f k8s/virtual-server.yaml
```

### Prepare the custom Helm Values File

We need to make necessary edits consistent to your Kubernetes environment and private container registry

1. From the project root, find the custom NGINX Ingress helm values file (`values.yaml`) in the `helm` folder:

```bash
cd helm
ls

values.yaml
```


#### Add correct `repository` uri

Make sure to specify the correct uri to your NGINX Ingress container. Configuring access to your private repository is NOT covered here, however, see [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) for guidance. In this example my EKS has access to the ECR where my docker images are stored: 

1. Modify `values.yaml` and include the correct uri to your container image or NGINX ingress (note the `tag` specifies we are using out customer NJS NGINX ingress image):

```yaml
# values.yaml

controller:
  nginxplus: true
  image:
    repository: myregistry.com/nginx-ingress
    tag: "1.9.0-njs"
#...
```

### Configure the correct DNS resolver

To enable service discovery for Services and Pods in NGINX we need to point a resolver to the Kubernetes cluster DNS (`CoreDNS` or its precursor, `kube-dns`). 

If you are using a Managed Kubernetes platform you should be able to find the Kubernetes DNS service IP address from the platform's management console or dashboard. Nevertheless, you there are other ways to find the the DNS IP address, including using a utility pod deployed inside your cluster.

Notes taken from [Debugging DNS Resolution](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

1. We can deploy a `dnsutil` pod to confirm Kubernetes cluster DNS IP address: 

```
# Use this manifest to create our dnsutil Pod:
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml

pod/dnsutils created

# Verify its status
kubectl get pods dnsutils
NAME      READY     STATUS    RESTARTS   AGE
dnsutils   1/1       Running   0          <some-time>

```

2. Now run a command in that pod to retrieve that information. As you can see in the example below the IP address is `10.100.0.10`:

```
kubectl exec -i -t dnsutils -- nslookup kubernetes.default

Server:         10.100.0.10
Address:        10.100.0.10#53

Name:   kubernetes.default.svc.cluster.local
Address: 10.100.0.1
```

3. Update Helm values file with the correct `resolver-addresses` value in the `values.yaml` file

```
controller:
  nginxplus: true
  image:
    repository: myregistry.com/nginx-ingress
    tag: "1.9.0-njs"
  config:
    entries:
      anykey: "1"
      resolver-addresses: "10.100.0.10"
      #etc...
```

4. Remove the utility pod when finished

```bash
Kubectl delete pod dnsutils
```

### Deploy IngressClass

For `1.18` and higher, we now require deploying an `ingressClass` resource. As of writing this guide (`11/nov/2020`) the `IngressClass` spec is missing from the helm chart and we must deploy this manually. You can find the `IngressClass` manifest on NGINX Plus Ingress repo, [here](https://github.com/nginxinc/kubernetes-ingress/blob/v1.9.0/deployments/common/ingress-class.yaml). For this demo we have included the `ingressClass` for the **NGINX Plus Ingress 1.9.0** at `k8s/ingress-class.yaml`

An example error message after running `helm install` where `ingressClass` is missing:

```bash
I1108 07:54:07.148530       1 main.go:245] Starting NGINX Ingress controller Version=1.9.0-njs GitCommit=de1c27ce
W1108 07:54:07.167454       1 main.go:284] The '-use-ingress-class-only' flag will be deprecated and has no effect on versions of kubernetes >= 1.18.0. Processing ONLY resources that have the 'ingressClassName' field in Ingress equal to the class.
F1108 07:54:07.173460       1 main.go:288] Error when getting IngressClass nginx: ingressclasses.networking.k8s.io "nginx" not found

```

We need to deploy the `ingressClass` to our cluster to prevent that error  (`11/nov/2020`):

1. First, check if `ingressClass` is deployed

```bash
# ingressClass not found:
kubectl get ingressclass -A

No resources found
```


2. In the case helm did not deploy it we can deploy it manually

```
# Using the known ingressClass for NGINX plus 1.9.0 
kubectl apply -f k8s/ingress-class.yaml

ingressclass.networking.k8s.io/nginx created
```

## Install NGINX Plus Ingress using Helm (local chart source, after template modifications)

We are following Instructions taken from the official [NGINX Plus Helm Installation Guide](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/) and assuming you are using a **Helm 3.x client**

**Note:** We will install the helm chart for demonstration purposes, but as of writing this guide (`11/nov/2020`) we need to modify the helm template an add `lifecycle` `preStop` hook for better zero downtime support. Details of that [here](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

### Using Helm 3.x client
1. First make sure you have **Helm 3.x** installed on your client machine (See [Helm installation guide](https://helm.sh/docs/intro/install/))

```bash
# On Mac OS
brew install helm

# Check you have a helm 3.x client
helm version
version.BuildInfo{Version:"v3.4.0", GitCommit:"7090a89efc8a18f3d8178bf47d2462450349a004", GitTreeState:"dirty", GoVersion:"go1.15.3"}
```

### Download and modify Helm Chart Sources

To enable better zero downtime support of our NGINX Plus Ingress Deployment we need to modify the helm template an add `lifecycle` `preStop`  into the `controller-deployment.yaml` file:

1. Download the Chart Sources

```bash
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress/deployments/helm-chart
git checkout v1.9.0
```

2. Adding the Helm Repository (to show but not used for this demo since we are using a custom Deployment not included in the latest Helm release as of writing `11/nov/2020`)

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
```

3. Locate `controller-deployment.yaml` inside the `helm-chart` folder in the `kubernetes-ingress` project folder and add in the following lines under the `containers -images` values:

```bash
cd kubernetes-ingress/deployments/helm-chart 
ls

Chart.yaml       chart-icon.png   templates        values-plus.yaml
README.md        crds             values-icp.yaml  values.yaml
```

These are the new specs we want to add to `controller-deployment.yaml`

```yaml
        lifecycle:
          preStop:
            exec:
              command:
              - sleep
              - "10"
```

Under `containers - image` i.e. 

```yaml
# controller-deployment.yaml

		# etc..
      containers:
      - image: "{{ .Values.controller.image.repository }}:{{ .Values.controller.image.tag }}"
        lifecycle:
          preStop:
            exec:
              command:
              - sleep
              - "10"
```

### Install NGINX Ingress using  Helm and local Chart Sources

Now all preparation of container images, deploying backend services, the missing `ingressClass`, `configmap` for our NJS code and `virtualServer` are deployed on our Kubernetes cluster.  We can finally deploy NGINX ingress using helm

1. Now install NGINX Plus Ingress Controller using Helm into our designated `nginx-ingress` namespace. We will need to specify the path to the chart directory and custom values. This will deploy NGINX with our deployment configs:
   * Helm Values (`values.yaml`)
     * Using the custom Docker image with NJS installed
     * DNS Resolver in our cluster
     * Deployment`replicaCount:` 3
     * Including all necessary NGINX.conf extensions using snippets and  volume mount for configMaps
   *  configMaps (`njs-configmap.yaml`) with our NGINX NJS code specific for this demo use case
   * The modified deployment spec (`nginx-plus-ingress.yaml`) with the `lifecycle preStop`

```bash
# We can do a dry run first. Append `--dry-run --debug` to only test:
helm install my-release -f helm/values.yaml kubernetes-ingress/deployments/helm-chart 
 --dry-run --debug --namespace nginx-ingress

# And for reals now:
helm install my-release -f helm/values.yaml kubernetes-ingress/deployments/helm-chart --namespace nginx-ingress
```

4. Wait for pods and services to get created:

```bash
kubectl get service,pod

NAME                                           READY   STATUS    RESTARTS   AGE
pod/my-release-nginx-ingress-84d7c75d7-f6tlw   1/1     Running   0          22m
pod/my-release-nginx-ingress-84d7c75d7-twvc7   1/1     Running   0          22m
pod/my-release-nginx-ingress-84d7c75d7-zzbsn   1/1     Running   0          22m

NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                      AGE
service/my-release-nginx-ingress   LoadBalancer   10.100.157.48   af30066a68d2c48c58f8829805ea8224-708974963.us-east-2.elb.amazonaws.com   80:30586/TCP,443:31781/TCP   37m
```

5. See it with `helm ls`

```bash
helm ls --namespace nginx-ingress

NAME            NAMESPACE       REVISION        UPDATED                                STATUS   CHART                   APP VERSION
my-release      nginx-ingress   1               2020-11-13 16:44:13.846383259 -0700 MSTdeployed nginx-ingress-0.7.0     1.9.0
```



**Note:** Because my Cloud Environment (AWS EKS) allows for it,  a Cloud `LoadBalancer` has been deployed to route External traffic to our NGINX ingress deployments. Make sure the necessary  security settings have been configured to allow external traffic to enter your Kubernetes cluster



## Testing our NGINX Ingress and Zero-downtime upgrades 

How that everything is deployed, let's try doing a live update!

See `demo.md` for upgrade examples



# EXTRA

## Deploy Cloud LoadBalancer

If your cloud load balancer is not deployed automaticly, you can deploy it yourself. Each environement and cloud platform is differerent. See instructions below as an example only

1. Get labels for ingress pod

```bash
kubectl get pods --namespace nginx-ingress --show-labels

NAME                                       READY   STATUS    RESTARTS   AGE   LABELS
my-release-nginx-ingress-f874cff86-27sg8   1/1     Running   0          57s   app=my-release-nginx-ingress,pod-template-hash=f874cff86

kubectl get pods --namespace nginx-ingress -lapp=my-release-nginx-ingress
NAME                                       READY   STATUS    RESTARTS   AGE
my-release-nginx-ingress-f874cff86-27sg8   1/1     Running   0          85s

```

3.  Add correct label to the cloud loadbalancer manifest `loadbalancer-aws-elb.yaml`


```yaml
#loadbalancer-aws-elb.yaml

#etc..
  selector:
    app: my-release-nginx-ingress

```

4. Deploy `loadBalancer`


```bash
kubectl apply -f k8s/loadbalancer-aws-elb.yaml

```

## Troubleshooting

Application Introspection and Debugging using `kubectl describe pod` to fetch details about pods 


```bash
# kubectl describe pod [pod] --namespace [namespace]

kubectl describe pod my-release-nginx-ingress-f874cff86-ndllz --namespace nginx-ingress
```

Debug Running Pods using `kubectl logs`

```bash
# kubectl logs [pod] -p --namespace [namespace]

kubectl logs my-release-nginx-ingress-f874cff86-lxsxg -p --namespace nginx-ingress
```

Debugging with container `exec`

```bash
# kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}

# Output the translated nginx.conf on the NGINX Ingress pod
kubectl exec -i -t my-release-nginx-ingress-f874cff86-lxsxg --namespace nginx-ingress -- nginx -T

# Run a nslookup in a dnsutil pod
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```


## Clean up

We can clean up all the deployed NGINX ingress, pods and services in this demo

1. Remove NGINX Ingress using `helm uninstall`

```bash
helm uninstall my-release --namespace nginx-ingress

release "my-release" uninstalled
```

2. Remove backend services

```bash
# Find the service and pod names
kubectl  get service,pod
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/dev-backend-svc    ClusterIP   10.100.220.35   <none>        80/TCP    3d9h
service/kubernetes         ClusterIP   10.100.0.1      <none>        443/TCP   3d17h
service/prod-backend-svc   ClusterIP   10.100.2.105    <none>        80/TCP    3d9h
service/vinrouter-svc      ClusterIP   None            <none>        80/TCP    3d9h

NAME                                READY   STATUS    RESTARTS   AGE
pod/dev-backend-65d795b777-gx8b2    1/1     Running   0          3d9h
pod/prod-backend-848bc9c989-5qpkx   1/1     Running   0          3d9h
pod/vinrouter-64878bfcb4-bp9lp      1/1     Running   0          3d9h

# Delete pods
kubectl delete pod dev-backend-...
kubectl delete pod prod-backend-...
kubectl delete pod vinrouter-...
kubectl delete service/dev-backend-svc
kubectl delete service/prod-backend-svc
kubectl delete service/vinrouter-svc
kubectl delete configmaps/njsv1
kubectl delete configmaps/njsv2
kubectl delete configmaps/njsv...
kubectl delete virtualServer/demo-example
```

