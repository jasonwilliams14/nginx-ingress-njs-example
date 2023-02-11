# Demo time

Now that everything is deployed, let's try doing a live updates and test Zero Downtime deployments

## Demo preparation 
1. Show backend pods and service in the default namespace

```bash
kubectl get service,pod

NAME                                           READY   STATUS    RESTARTS   AGE
pod/my-release-nginx-ingress-84d7c75d7-f6tlw   1/1     Running   0          22m
pod/my-release-nginx-ingress-84d7c75d7-twvc7   1/1     Running   0          22m
pod/my-release-nginx-ingress-84d7c75d7-zzbsn   1/1     Running   0          22m

NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                      AGE
service/my-release-nginx-ingress   LoadBalancer   10.100.157.48   af30066a68d2c48c58f8829805ea8224-708974963.us-east-2.elb.amazonaws.com   80:30586/TCP,443:31781/TCP   37m

```

2. If you have NGINX ingress deployed using helm, first delete it with helm remove uninstall 

```bash
helm ls --namespace nginx-ingress

NAME            NAMESPACE       REVISION        UPDATED                                STATUS   CHART                   APP VERSION
my-release      nginx-ingress   4               2020-11-13 16:44:13.846383259 -0700 MSTdeployed nginx-ingress-0.7.0     1.9.0
N
```
3. Uninstall NGINX ingress

```bash
helm uninstall my-release --namespace nginx-ingress
 
helm ls --namespace nginx-ingress
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
```


## Demo script

### Optional: Review the manifests and documentation

1. Enter project root

```bash
ls
apps demo.md  helm  k8s  kubernetes-ingress  nginx-plus-ingress  README.md
```

2. Quickly show the demo services artifacts

```bash
# Show the backend service and pods
code k8s/backends.yaml

# Show the vinrouter service and pods
code k8s/vinrouter.yaml
```

3. Review  the helm and NGINX artifacts

Helm, `virtualServer` and `VirtualServerRoute` Documentation:
```bash
# Read helm documentaion
firefox https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/

# Read virtualServer and VirtualServerRoute resources
firefox https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/
```

To improve  Zero downtime deployments of NGINX ingress without any downtime we can add
a `lifecycle` `preStop` with a `sleep` command to give Kubernetes enough time to remove the pod from the service (so no lost requests). This prevents a race condition where Kubernetes marks pods as terminating but takes a little time (< 1s?) to remove this pod from the service:

Review the  modified NGINX Deployment Pod spec in the helm chart source:

```yaml
#kubernetes-ingress/deployments/helm-chart/templates/controller-deployment.yaml

        lifecycle:
          preStop:
            exec:
              command:
              - sleep
              - "10"
```

Review the nesscary NGINX.conf snippets in the the following files:

```bash

# helm values
# loading NJS Module and NGINX logic  to achieve subrequest calls
code helm/values.yaml

# NJS code configMap 
# NJS logic to achieve subrequest calls
code k8s/njs-configmap.yaml

# virtualserver
# some NGINX logic to achieve subrequest calls and 
# demo.example.com service and routing rules
code k8s/virtual-server.yaml
```

### Deploy NGINX Ingress using Helm!

1. Deploy NGINX Ingress using Helm!

```bash
helm install my-release -f helm/values.yaml kubernetes-ingress/deployments/helm-chart --namespace nginx-ingress
```

2. Show the ingress is running, the replica count and the default loadBalancer service created

```bash
watch kubectl get pods,services --namespace nginx-ingress


NAME                                           READY   STATUS    RESTARTS   AGE
pod/my-release-nginx-ingress-84d7c75d7-9kfwc   1/1     Running   0          15s
pod/my-release-nginx-ingress-84d7c75d7-bpgmf   1/1     Running   0          15s
pod/my-release-nginx-ingress-84d7c75d7-x8n7t   1/1     Running   0          15s

NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                      AGE
service/my-release-nginx-ingress   LoadBalancer   10.100.42.235   a979aa5ab2fb94de6a771f69a08df3d2-576122314.us-east-2.elb.amazonaws.com   80:32344/TCP,443:32022/TCP   16s
```

3. We can also see the deployment using `helm ls`

```bash
helm ls -n nginx-ingress

NAME            NAMESPACE       REVISION        UPDATED                                STATUS   CHART                   APP VERSION
my-release      nginx-ingress   1               2020-11-13 17:19:04.97493155 -0700 MST deployed nginx-ingress-0.7.0     1.9.0

```

### Test our NGINX Ingress services

### Find the external FQDN of our `LoadBalancer`

1. Get the external FQDN of our `LoadBalancer`. In the example below the FQDN is `aea0beb5aeb844c51adc7662dd7f018e-346780955.us-east-2.elb.amazonaws.com`:

```bash
kubectl get svc -n=nginx-ingress

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                      AGE
my-release-nginx-ingress   LoadBalancer   10.100.98.213   aea0beb5aeb844c51adc7662dd7f018e-346780955.us-east-2.elb.amazonaws.com   80:30489/TCP,443:31713/TCP   10m
```

OR more useful is to save the External LoadBalaner address 

```bash
EXTERNAL_LB=$(k get services -n nginx-ingress | awk {'print $4 '} | grep elb.)
echo $EXTERNAL_LB
```

To use that External Address in `curl` we need the translated IP Address:

```
dig +short $EXTERNAL_LB A
18.218.153.11
18.216.190.149
3.129.154.135

IP_EXTERNAL_LB=$(dig +short $EXTERNAL_LB A |  awk 'NR==1')
echo $IP_EXTERNAL_LB
```

We will use the FQDN or its translated IP Address (`$IP_EXTERNAL_LB`) in the following test...

### Test request to Dev and  Prod backends

Lets make requests to our services!

The following NGINX.conf snippet in our test `vinrouter` routes request to the  **"Dev" backend**  when the `CN` value in the `CCRT-Subject` HTTP Header begins with a number and routes request to the  **"Dev" backend**   when the  `CN` value HTTP Header begins with a letter or anything other character:

```nginx
# vinrouter/router.conf
map $request_uri $env {
    default   PROD;     
    ~/api/vins/([0-9].*) DEV;  # not empty
}
```

####  Prod backends

1. Run `curl` using the `--resolve` binding the External LoadBalancer IP to the expected hostname `demo.example.com`,  we expect requests to route to `production`

```bash
# Use curl with the --resolve parameter to resolve demo.example.com to the IP address of out loadBalancer

curl  http://demo.example.com/web-services/user-data/1.1/auto-get-profiles-timestamp -H 'CCRT-Subject: C=DE, O=Daimler AG, OU=MBIIS-CERT, CN=ADF4477' --resolve demo.example.com:80:$IP_EXTERNAL_LB

{
	"environment": "production"
}
```

#### Test Dev

1. Run `curl` using the `--resolve` binding the External LoadBalancer IP to the expected hostname `demo.example.com`, this time we expect requests to route to `dev`

```bash
# Use curl with the --resolve parameter to resolve demo.example.com to the IP address of out loadBalancer

curl  http://demo.example.com/web-services/user-data/1.1/auto-get-profiles-timestamp -H 'CCRT-Subject: C=DE, O=Daimler AG, OU=MBIIS-CERT, CN=0DF4477' --resolve demo.example.com:80:$IP_EXTERNAL_LB
{
            "environment": "dev"
}
```

## Testing Zero downtime

As we run through these tests, we can view the NGINX pods and services in our
designated `nginx-ingress` namespace to confirm the `AGE` is not restarted to `0` and existing pods are NOT terminated

```bash
watch kubectl get pods,services --namespace nginx-ingress

NAME                                           READY   STATUS    RESTARTS   AGE
pod/my-release-nginx-ingress-84d7c75d7-9kfwc   1/1     Running   0          8m12s
pod/my-release-nginx-ingress-84d7c75d7-bpgmf   1/1     Running   0          8m12s
pod/my-release-nginx-ingress-84d7c75d7-x8n7t   1/1     Running   0          8m12s

NAME                               TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                      AGE
service/my-release-nginx-ingress   LoadBalancer   10.100.42.235   a979aa5ab2fb94de6a771f69a08df3d2-576122314.us-east-2.elb.amazonaws.com   80:32344/TCP,443:32022/TCP   8m13s

```

### Scaling backend services - illustrated using NGINX dashboard

In a separate terminal, use Port Forwarding to Access the NGINX Plus dashboard in a Cluster. In this example the dashboard is listening on port `8080`

NGINX Plus required **No reload** upon reconfigurations of upstreams

1. Run `kubectl port-forward` to view the NGINX Plus dashboard in on your `localhost` address

```bash
NGINX_INGRESS_1=$(kubectl get pods -n nginx-ingress | grep nginx-ingress | awk {'print $1 '} | awk 'NR==1')

# kubectl port-forward <nginx-ingress-pod> 8080:8080 --namespace=[namespace]
kubectl port-forward $NGINX_INGRESS_1 8080:8080 --namespace=nginx-ingress
```

2. Then open in a web browser:

```bash
firefox http://localhost:8080/dashboard.html
```

3. Scale the coffee ReplicaSet up and down and notice the changes in the upgream groups (see `HTTP Upstreams`):

```bash
kubectl scale deployment dev-backend  --replicas=4
kubectl scale deployment prod-backend --replicas=4
```

4. You can also confirm the scaling services by running the command:

```bash

kubectl get pods,services


NAME                                READY   STATUS    RESTARTS   AGE
pod/dev-backend-65d795b777-fsns2    1/1     Running   0          3h39m
pod/dev-backend-65d795b777-gl5df    1/1     Running   0          58m
pod/dev-backend-65d795b777-nmnjx    1/1     Running   0          58m
pod/dev-backend-65d795b777-tx558    1/1     Running   0          58m
pod/prod-backend-848bc9c989-2q7k5   1/1     Running   0          57m
pod/prod-backend-848bc9c989-5qpkx   1/1     Running   0          5d17h
pod/prod-backend-848bc9c989-hk57g   1/1     Running   0          57m
pod/prod-backend-848bc9c989-qdnmj   1/1     Running   0          57m
pod/vinrouter-64878bfcb4-bp9lp      1/1     Running   0          5d17h

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/dev-backend-svc    ClusterIP   10.100.220.35   <none>        80/TCP    5d17h
service/kubernetes         ClusterIP   10.100.0.1      <none>        443/TCP   6d1h
service/prod-backend-svc   ClusterIP   10.100.2.105    <none>        80/TCP    5d17h
service/vinrouter-svc      ClusterIP   None            <none>        80/TCP    5d17h

```

### Helm upgrades - no downtime

1. Watch the nginx-ingress pods

```bash
watch kubectl get pods -n nginx-ingress
```

#### Update virtualServer config - NGINX graceful reload only

Updates to  `virtualServer` and `virtualServerRoutes`  require a  **Graceful reload** of NGINX, with **No Downtime**

Addtionally updates to `virtualServer` and `virtualServerRoutes` do not require a `helm upgrade`

1. Make modificaitons to the virtual server CRD `virtual-server.yaml`, and be careful not to break the `yaml` syntax

e.g. Add a comment in the `http-snippets` section in `virtual-server.yaml`

```yaml
# virtual-server.yaml
    etc..
    http-snippets: |
        # Hello!
    etc...
```

2. Then apply the config

```bash
k apply -f k8s/virtual-server.yaml
```

3. Check that the changes have been propogates to NGINX. Confirm by viewing the NGINX.config translated by the NGINX controller:

```bash

k exec -it pod [pod name] -n nginx-ingress -- nginx -T | grep " Hello!"
```
4. Notice NGINX pods are not terminated, nor are new pods spawned. NGINX only did a graceful reload

#### Update Helm Values  - NGINX graceful reload only


1. Make modifications to the helm values `values.yaml`, and be careful not to break the `yaml` syntax

e.g. Add a comment in the `http-snippets` section in helm values file `values.yaml`, and be careful not to break the `yaml` syntax

```yaml
# values.yaml
    etc..
    http-snippets: |
        # Hello!
    etc...
```

2. Then apply changes using `helm upgrade`

```bash
helm upgrade my-release -f helm/values.yaml kubernetes-ingress/deployments/helm-chart --namespace=nginx-ingress
```

3. Check for NGINX config translates by the NGINX controller in the nginx pods:

```bash
k exec -it pod [pod name] -n nginx-ingress -- nginx -T | grep " Hello!"
```

4. Notice NGINX pods are not terminated, nor are new pods spawned. NGINX only did a graceful reload

#### Update NJS code configMap - New NGINX pod deployement NGINX (new pods) with zero downtime

Helm is a tool to help you deploy your application to kubernetes. It doesn't replace any kubernetes functionality. Zero downtime deployments are provided by Kubernetes with use of the deployment type, rollout strategy, and liveness + readiness probes. See [kubernetes.io/docs/concepts/workloads/controllers/deployment/… – Curtis Allen Nov 30 '18 at 19:09](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)

If you change those to Deployments, Kubernetes will automatically manage replication and zero-downtime updates for you:

1. Make modificaitons to the NJS configmap, `njs-configmap.yaml`, and be careful not to break the `yaml` syntax

e.g. Add a Javascript comment ("`//hello!`") and provide a new name  ("`njsv2`")  that will be treated as a new configMap:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: njsv2
  namespace:  -nnginx-ingress
data:
  vinrouter.js: |

    //hello!
```

2. Deploy the configMap

```bash
kubectl apply -f k8s/njs-configmap.yaml -n nginx-ingress
```

3. We also need update the corresponding a `configMap` `name` in the Helm values file, `values.yaml`:

For example update the key
```yaml
 #...
  - name: njs
    configMap:
      name: njsv2
```

**before we perform a `helm upgrade` run some load...**

4. Now run traffic using a load generator to simulate live traffic:

Get the External Load Balancer address again

```bash
EXTERNAL_LB=$(k get services -n nginx-ingress | awk {'print $4 '} | grep .elb.)
echo $LB
IP_EXTERNAL_LB=$(dig +short $EXTERNAL_LB A |  awk 'NR==1')
echo $IP_EXTERNAL_LB
```

Start the [`vegeta`](https://github.com/tsenart/vegeta) load generator for `90s` ( It's best to let the load generator end on its own so don't run the test for too long)

```bash
echo "GET http://$IP_EXTERNAL_LB/web-services/user-data/1.1/auto-get-profiles-timestamp" \
    | vegeta attack -duration=90s \
        -header "CCRT-Subject: C=DE, O=Daimler AG, OU=MBIIS-CERT, CN=ADF4477" \
        -header "Host: demo.example.com" \
    | tee results.bin | vegeta report
```

4. **While the load test is running**, perform a `helm upgrade`

```bash
helm upgrade my-release -f helm/values.yaml kubernetes-ingress/deployments/helm-chart --namespace=nginx-ingress
```

5. Notice the old pods terminating and new pods spawning

6. Allow the  `vegeta`  Load generator to complete its test duration. If needed (but not recommended)  you can stop the `vegeta` load test with `Ctrl+C`

7. In the test summary output, confirm there were no dropped requests during the upgrade. Notice NGINX pods are  terminated, and new pods spawned. Old NGINX served out all requests before terminating and there should be no connections droped




