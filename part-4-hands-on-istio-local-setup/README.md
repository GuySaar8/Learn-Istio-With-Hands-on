# Part 4: Hands-On with Istio Local Setup Using k3d

![Istio Lighthouse](./image.png)

---

This is the fourth article in the Understanding Service Mesh series. So far, we've covered:
- What are service meshes, and why do they matter
- We covered one of the most widely used service mesh implementations: Istio, and its core components like istiod, Ingress Gateway, and sidecar proxies
- How do control and data planes work together in the mesh

Now it's time to put that knowledge into action.
This hands-on guide will walk you through installing Istio locally, deploying a sample app, and routing HTTP traffic through the Istio Gateway all using a local Kubernetes cluster powered by k3d.
By the end, you'll be able to visit your app in a browser via http://local.test.
This will be the base start of any of the following hands-on articles.

---

## What We'll Do
- Spin up a local Kubernetes cluster with k3d
- Install Istio with Helm
- Deploy a test app
- Route browser traffic to it using Istio Gateway and VirtualService
- Access it from http://local.test

Let's get started.

---

## Prerequisites
Before you begin, ensure your local environment meets the following requirements:
- Docker installed and running.
- k3d installed to run Kubernetes clusters locally.
- kubectl command-line tool installed and configured to access your cluster.
- Helm installed for managing Kubernetes packages (used for Istio installation).
- A local machine with sufficient memory resources to run Istio. Istio typically requires at least 4GB of memory allocated per server and agent node to operate smoothly.

Make sure your environment can meet these requirements to avoid performance issues during the setup.

---

## Step 1: Create a Local k3d Cluster
Make sure Docker is running
k3d spins up Kubernetes nodes as Docker containers, so the Docker engine must be active on your machine.

```sh
docker info
```
If you're getting an error, please go check: https://www.docker.com/products/docker-desktop
If that command returns information instead of an error, Docker is up and you're ready to create the cluster.

Let's create our local Kubernetes cluster using k3d, we will expose port 80 (HTTP) and port 443 (HTTPS) on your machine and disable Traefik.

```sh
k3d cluster create istio-demo \
  --agents 1 \
  --port 8080:80@loadbalancer \
  --port 8443:443@loadbalancer \
  --k3s-arg '--disable=traefik@server:0' \
  --servers-memory 4g \
  --agents-memory 4g

kubectl config use-context k3d-istio-demo
```

This gives us:
- A lightweight Kubernetes cluster
- Port 80 mapped to the Istio ingress gateway
- A clean slate without other ingress controllers

**Verify the Cluster is Running:**
After creating the cluster, ensure it's up and running by checking the available namespaces:

```sh
kubectl get pod -n kube-system
```
You should see output similar to:
```
NAME                                      READY   STATUS    RESTARTS   AGE
coredns-ccb96694c-hhx22                   1/1     Running   0          27s
local-path-provisioner-5cf85fd84d-wmqc5   1/1     Running   0          27s
metrics-server-5985cbc9d7-z9rcb           1/1     Running   0          27s
```

---

## Step 2: Install Istio Using Helm
First, add the Istio chart repository:

```sh
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Now install Istio's components:

```sh
helm install istio-base istio/base -n istio-system --create-namespace
helm install istiod istio/istiod -n istio-system --wait
helm install istio-ingressgateway istio/gateway -n istio-system --wait
```

- `istio-base` sets up Istio's Custom Resource Definitions (CRDs) required for the other components to function.
- `istiod` is Istio's control plane, responsible for managing configuration, pushing policies, handling certificate issuance, and injecting sidecars into application pods.
- `istio-ingress` deploys an Ingress Gateway, which is the main entry point for external traffic into your service mesh.

**Verify Istio Installation:**
Once Istio is installed, make sure everything is running correctly:

```sh
kubectl get all -n istio-system
```
You should see resources like istiod and istio-ingressgateway in a Running state, along with associated services:

Example output:
```
NAME                                           READY   STATUS    RESTARTS   AGE
pod/istiod-xxxxxxxxxx-xxxxx                    1/1     Running   0          30s
pod/istio-ingressgateway-xxxxxxxxxx-xxxxx      1/1     Running   0          30s

NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
service/istiod                    ClusterIP      10.43.0.1      <none>        15010/TCP,15012/TCP  30s
service/istio-ingressgateway      LoadBalancer   10.43.0.2      localhost     80:31380/TCP         30s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istiod                  1/1     1            1           30s
deployment.apps/istio-ingressgateway    1/1     1            1           30s
```
Update Complete. ⎈Happy Helming!⎈

![⎈Happy Helming!⎈](./image2.png)

---

## Step 3: Deploy a Sample App
We'll deploy a simple NGINX application in Kubernetes. The key here is enabling Istio sidecar injection by labeling the namespace accordingly.

Create a file named `app.yaml` with the following resources:
- **Namespace:** `demo` labeled with `istio-injection: enabled` to automatically inject Istio sidecars into pods in this namespace.
- **Deployment:** A single replica running the NGINX container exposing port 80.
- **Service:** A ClusterIP service named `web` that targets the NGINX pods on port 80.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  labels:
    istio-injection: enabled
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

Apply it:
```sh
kubectl apply -f app.yaml
```

**Verify the deployment and service are running correctly:**
```sh
kubectl get all -n demo
```
You should see output like this:
```
NAME                       READY   STATUS    RESTARTS   AGE
pod/web-85ccb5df5b-d46pv   2/2     Running   0          20s

NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/web   ClusterIP   10.43.101.184   <none>        80/TCP    20s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   1/1     1            1           20s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/web-85ccb5df5b   1         1         1       20s
```
Notice the pod's READY status shows 2/2 instead of 1/1 this is because the Istio sidecar proxy container has been injected alongside the NGINX container, resulting in two containers running within the pod.

---

## Step 4: Configure Istio Gateway and VirtualService

### What Are Gateway and VirtualService?
- **Gateway:** An Istio resource that defines how external traffic enters the mesh like opening specific ports on the edge and handling protocols such as HTTP or HTTPS.
- **VirtualService:** Defines routing rules for traffic once it's inside the mesh for example, which service to send it to, how to match paths, apply timeouts, retries, etc.

Together, they let you control what comes in and where it goes.

### Defining the Gateway and VirtualService
Create a `gateway.yaml` file with:

**Gateway**
- It selects Istio's built-in ingress gateway via the label `istio: ingressgateway`.
- Opens port 80 for HTTP traffic.
- Restricts traffic to the host `local.test`.

**VirtualService**
- Routes requests matching host `local.test`.
- Associates with the `web-gateway`.
- Routes traffic to the NGINX service (`web.demo.svc.cluster.local`) on port 80.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - local.test
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web
  namespace: demo
spec:
  hosts:
  - local.test
  gateways:
  - istio-system/web-gateway
  http:
  - route:
    - destination:
        host: web.demo.svc.cluster.local
        port:
          number: 80
```

Apply it:
```sh
kubectl apply -f gateway.yaml
```

---

**Verify the gateway and the virtual service:**
```sh
kubectl get gateway,virtualservice -n demo
```
You should see output like this:
```
NAME                                      AGE
gateway.networking.istio.io/web-gateway   2m

NAME                                     GATEWAYS          HOSTS            AGE
virtualservice.networking.istio.io/web   ["web-gateway"]   ["local.test"]   2m
```

## Step 5: Map local.test to Your Machine
To simulate DNS resolution for your custom domain (`local.test`), you need to manually map it to your local machine. This is done by adding an entry to your `/etc/hosts` file, which acts like a simple, local DNS resolver.

Run the following command to map the domain to 127.0.0.1 (localhost):
```sh
echo "127.0.0.1 local.test" | sudo tee -a /etc/hosts
```
The `/etc/hosts` file allows you to override DNS lookups by specifying IP-to-hostname mappings manually. It's a useful tool for local development and testing with custom domains without needing a real DNS server.

---

## Step 6: Test in the Browser
Open your browser and visit:
```
http://local.test:8080
```
You should see the default NGINX welcome page served through Istio's Ingress Gateway.

---

## Clean-Up
Once you are done, clean up your local machine by running:
```sh
k3d cluster delete istio-demo
```

---

## Recap
You've now deployed:
- A local k3d cluster
- Istio via Helm
- A test app inside the mesh
- An Istio Gateway and VirtualService to expose it

Most importantly, you've routed real HTTP traffic from your browser through the mesh.

---

## What's Next?
Now that Istio is up and running locally, it's time to explore how Istio manages traffic using its powerful custom resources. In the next article, we'll dive into Gateways and get hands-on with managing inbound traffic including how to secure it with TLS termination.

Stay tuned! 