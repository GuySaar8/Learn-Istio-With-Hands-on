# Part 5: Istio Gateway Controlling Traffic at the Edge

![Istio Gateway Visual](./image.png)

---

In the previous posts of this series, we explored the fundamentals of service meshes, dove into Istio's architecture, and set up a local Istio environment using k3d, Helm, and an HTTP Gateway. Now that our mesh is up and running, it's time to talk about how traffic enters our mesh: the Istio Gateway.
This post breaks down what an Istio Gateway does, why it's important, and walks you through its key features in depth including hands-on experience.

---

## What Is an Istio Gateway?
In Kubernetes, Ingress resources provide basic routing for external traffic into your cluster. Istio offers a more powerful and flexible alternative: the Istio Gateway.

An Istio Gateway is a resource that configures the mesh's edge proxy (Envoy) to accept inbound or outbound traffic. It defines how traffic enters the mesh, not where it goes. Unlike Kubernetes Ingress, which combines edge handling and routing into one resource, Istio separates infrastructure from routing logic.

**What the Gateway does:**
- Exposes one or more ports (e.g., HTTP, HTTPS, TCP)
- Listens for traffic on specific hosts
- Terminates TLS, forwards traffic, or handles mTLS
- Selects which Envoy proxy this config applies to (via selector)

This separation of concerns makes Gateways composable, secure, and easier to manage in large environments.

---

## Istio Gateway
A Gateway resource is made up of three primary components: a selector, one or more servers, and optional TLS configuration.

### 1. Selector
The selector tells Istio which Ingress Gateway Deployment should implement the configuration set up in the Istio Gateway.

The selector is set up in the Gateway:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
```
It should match the labels set up in the Ingress Gateway Deployment and Service:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-ingressgateway
  namespace: istio-system
spec:
  template:
    metadata:
      labels:
        istio: ingressgateway
---
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
```
This is how Istio determines which controller "owns" or manages each Gateway configuration.

You can run multiple ingress gateways if needed (e.g., one for internal traffic, one for external), and apply different configurations to each.

### 2. Servers
Each server block defines a listener for incoming traffic. You can define multiple servers in a single Gateway resource.
A server contains:
- A port (number, name, and protocol)
- A list of allowed hostnames
- Optional TLS configuration

Example:
```yaml
servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
      - "example.com"
    tls:
      mode: SIMPLE
      credentialName: example-cert
```
This config tells Envoy to listen on port 443 for HTTPS traffic to example.com and terminate TLS using a certificate stored in a Kubernetes secret named example-cert.

---

## TLS Modes Explained
Istio supports several TLS modes in the Gateway configuration, each suited for different use cases:

### SIMPLE
In this mode, Istio terminates TLS at the gateway using the provided certificate and then forwards unencrypted HTTP traffic internally to your services.
This is the most typical setup for standard HTTPS simple, effective, and widely used in production when internal traffic doesn't need to remain encrypted.

### MUTUAL
In this MUTUAL mode, also known as mutual TLS (mTLS), adds an extra layer of security by requiring both the client and the server (in this case, the Istio Gateway) to authenticate each other with TLS certificates. Istio terminates TLS at the gateway, the same as the SIMPLE mode.

**How it works:**
- The Gateway holds the server's TLS certificate and private key, just like in SIMPLE mode, to terminate the TLS connection.
- When a client connects, the Gateway requests the client's certificate.
- The Gateway verifies the client's certificate against a trusted Certificate Authority (CA).
- Simultaneously, the client verifies the Gateway's certificate against its trusted CA to ensure it is connecting to the correct server.
- Only clients with valid, trusted certificates are allowed to establish a connection, and clients can be confident they are communicating with the authentic gateway.

### PASSTHROUGH
In PASSTHROUGH mode, Istio doesn't terminate TLS at all. It simply forwards the encrypted traffic as-is to the destination service, which is responsible for terminating TLS itself.
This is useful when your backend services are already set up to handle their own certificates and TLS logic for example, services using their own custom TLS stack or those running on platforms with built-in TLS.

### AUTO_PASSTHROUGH
AUTO_PASSTHROUGH builds on the PASSTHROUGH mode but adds intelligent, SNI-based routing. Instead of requiring explicit routing rules, Istio inspects the SNI (Server Name Indication) value in the TLS handshake to determine which service the traffic should go to.

This allows Istio to automatically forward encrypted traffic to the appropriate service without needing to define a VirtualService. It's especially powerful in multi-tenant or dynamically scaling environments, where manually managing routing rules would be complex and error-prone.

**Important:** While you don't need a VirtualService, you do need a DestinationRule for each service to enable mutual TLS or define traffic policies. Without it, traffic may not route correctly, especially in stricter mTLS configurations.

Use AUTO_PASSTHROUGH when you want the flexibility of dynamic routing but still need to maintain control over service-level policies using DestinationRules.

**When to Use What?**
- SIMPLE is the go-to choice for handling standard HTTPS traffic where TLS termination at the edge is sufficient.
- MUTUAL should be used when you need strong client authentication and encrypted communication both ways.
- PASSTHROUGH is appropriate when your backend services manage TLS themselves.
- AUTO_PASSTHROUGH works best for dynamic routing scenarios, especially when dealing with multiple backend services behind a single Gateway.

---

## What the Gateway Does Not Do
It's important to clarify what Istio Gateways are not responsible for:
- They do not define routing rules. That's the job of VirtualService, which we'll revisit in a future post.
- They do not select pods or perform load balancing. Envoy handles that internally after routing is determined.
- They do not automatically manage TLS certificates. You must provision them yourself (manually or via tools like cert-manager).

The Gateway is focused on edge traffic handling not service discovery, routing decisions, or backend logic.

---

## Why Istio Gateway Matters
Istio Gateway gives you explicit, secure, and scalable control over how traffic enters your mesh. With it, you can:
- Accept traffic only on specific ports and hostnames
- Terminate TLS securely and centrally
- Support advanced mTLS and passthrough configurations
- Cleanly separate infrastructure concerns from app-level routing

This structure becomes especially powerful when working in multi-tenant environments, or when exposing public APIs with strict security requirements.

---

## Hands-On: Configuring an Istio Gateway
Now that we've covered what an Istio Gateway is and why it matters, let's put that knowledge into practice.

In this section, you'll walk through configuring a Gateway and VirtualService to expose a test application over HTTPS, using the Istio Ingress Gateway we set up earlier. This hands-on example will show how traffic flows securely from your browser to a service inside the mesh all controlled by Istio's declarative configuration.

In the previous article, we exposed services through the Istio Gateway using plain HTTP. This time, we'll focus on HTTPS traffic and explore one of the TLS modes we covered earlier: SIMPLE.

![Istio Gateway TLS Visual](./image3.png)

---

### Scenario 1: HTTPS Gateway Using SIMPLE TLS Mode
We'll expose the NGINX service over HTTPS using a self-signed certificate.

#### Step 1: Generate a Self-Signed Cert
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout local-test.key -out local-test.crt \
  -subj "/CN=local.test" \
  -addext "subjectAltName = DNS:local.test"
```
Create a secret from this cert:
```sh
kubectl create -n istio-system secret tls local-test-tls \
  --key=local-test.key --cert=local-test.crt

kubectl label secret local-test-tls -n istio-system  istio.io/managed=true
```

#### Step 2: Define the Gateway
Update the Gateway in gateway.yaml file from supporting http to https
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway  # This targets the default Istio ingress gateway
  servers:
  - port:
      number: 443          # Listen on HTTPS port
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE         # TLS is terminated here
      credentialName: local-test-tls # Refers to the secret we just created
    hosts:
    - "local.test"  # Only allow this host
```
This tells the ingress gateway: "Listen on port 443 for HTTPS traffic to local.test. Use the secret local-test-tls to handle TLS."

#### Step 3: Define the VirtualService
Update the VirtualService in gateway.yaml file from supporting http to https
```yaml
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
        host: web.demo.svc.cluster.local # Route to the Kubernetes service named 'web'
        port:
          number: 80 # On port 80 (container port exposed in the service)
```
This says: "When the Gateway gets a request for local.test, route it to the web Kubernetes service on port 80."

#### Step 4: Test It
Apply the resources:
```sh
kubectl apply -f gateway.yaml
```
Then open your browser and visit:
```
https://local.test:8443
```
Since this uses a self-signed certificate, your browser will show a warning. Accept it to proceed. You should see the default NGINX welcome page served through Istio's Ingress Gateway over HTTPS.

---

### Scenario 2: Broken TLS Example
This scenario demonstrates how misconfigured TLS settings in the Gateway manifest lead to broken routing. It also shows where to look for debugging information (the ingress gateway logs), which is especially important when you start automating cert management or using more advanced TLS features.

#### Step 1: Simulate a Misconfigured TLS Secret
Edit your existing gateway.yaml and change the credentialName to a secret that doesn't exist:
```yaml
tls:
  mode: SIMPLE
  credentialName: wrong-secret  # Deliberately incorrect
```
Re-apply the Gateway resource:
```sh
kubectl apply -f gateway.yaml
```

#### Step 2: Test the Behavior
Now try visiting the app in your browser:
```
https://local.test:8443
```
You'll receive a connection failure or TLS handshake error, because the Gateway is attempting to terminate TLS using a secret that doesn't exist.

#### Step 3: Investigate the Problem
Check the logs of the Istio ingress gateway to troubleshoot:
```sh
kubectl logs -n istio-system -l istio=ingressgateway
```
Look for errors like:
```
warning envoy config external/envoy/source/extensions/config_subscription/grpc/grpc_subscription_impl.cc:130 gRPC config: initial fetch timed out for type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret thread=15
```
This is Envoy (via the ingress gateway) telling you that it cannot find the TLS secret referenced by the Gateway resource and therefore cannot start the listener.

Change back the configuration to match the right secret name to fix the issue
```sh
kubectl logs -n istio-system -l istio=ingressgateway
```

```
info Readiness succeeded in 29.434788622s
info Envoy proxy is ready
```

---

### Scenario 3: Adding Bookinfo over HTTPS (SIMPLE TLS Mode)
Now that you've exposed the NGINX app over HTTPS, let's add another service: Istio's sample Bookinfo app. We'll expose it over HTTPS using a new hostname: bookinfo.local.test.

#### Step 1: Deploy the Bookinfo Application
Apply the Bookinfo manifest:
```sh
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/bookinfo/platform/kube/bookinfo.yaml
```
Verify the pods:
```sh
kubectl get pods -n demo
```
Wait until all Bookinfo pods are running and show 2/2 (app + sidecar proxy).

#### Step 2: Create a Self-Signed TLS Certificate for bookinfo.local.test
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout bookinfo.key -out bookinfo.crt \
  -subj "/CN=bookinfo.local.test" \
  -addext "subjectAltName = DNS:bookinfo.local.test"
```
Create a TLS secret in the istio-system namespace:
```sh
kubectl create -n istio-system secret tls bookinfo-tls \
  --key=bookinfo.key --cert=bookinfo.crt

kubectl label secret bookinfo-tls -n istio-system  istio.io/managed=true
```

#### Step 3: Add a New Server to the Existing Gateway
Let's create a new file for the Gateway resource:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway  # This targets the default Istio ingress gateway
  servers:
  - port:
      number: 443          # Listen on HTTPS port
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE         # TLS is terminated here
      credentialName: bookinfo-tls # Refers to the secret we just created
    hosts:
    - "bookinfo.local.test"  # Only allow this host
```
Apply the new file bookinfo-gateway:
```sh
kubectl apply -f bookinfo-gateway.yaml
```

#### Step 4: Define a VirtualService for Bookinfo
Let's add another route for bookinfo to bookinfo-gateway file
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
  namespace: demo
spec:
  hosts:
  - bookinfo.local.test
  gateways:
  - istio-system/bookinfo-gateway
  http:
  - route:
    - destination:
        host: productpage.demo.svc.cluster.local
        port:
          number: 9080
```
Apply it:
```sh
kubectl apply -f bookinfo-gateway.yaml
```

#### Step 5: Update /etc/hosts
Add the following line to your /etc/hosts file to resolve bookinfo.local.test to your local machine:
```sh
echo "127.0.0.1 bookinfo.local.test" | sudo tee -a /etc/hosts
```

#### Step 6: Test It
In your browser, visit:
```
https://bookinfo.local.test:8443/productpage
```
You'll get a browser warning due to the self-signed cert, but once accepted, you should see the Bookinfo product page being served securely over HTTPS via the Istio Gateway.

---

### Scenario 4: Exposing Bookinfo over HTTPS with Mutual TLS (mTLS)
Now that Bookinfo is exposed via HTTPS, let's raise the security bar by enabling mutual TLS (mTLS). With mTLS, clients must also present a valid certificate signed by a trusted Certificate Authority (CA) not just the server. We'll continue using the bookinfo.local.test hostname from the previous scenario.

To better understand how TLS and certificate validation work under the hood, we'll stick with openssl commands in this example. This hands-on approach gives visibility into the mTLS handshake and trust relationships.

That said, in production environments, it's highly recommended to use a Kubernetes-native certificate manager like cert-manager to automate certificate issuance, renewal, and rotation.

#### Step 1: Generate Certificates for mTLS
We'll generate a self-signed CA, and use it to sign both server and client certificates.

Create a self-signed CA:
```sh
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout ca.key -out ca.crt \
  -subj "/CN=bookinfo-ca"
```
Generate and sign the server certificate:
```sh
openssl req -newkey rsa:2048 -nodes -keyout bookinfo-server.key -out bookinfo-server.csr \
  -subj "/CN=bookinfo.local.test" \
  -addext "subjectAltName = DNS:bookinfo.local.test"

openssl x509 -req -in bookinfo-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out bookinfo-server.crt -days 365
```
Create a Kubernetes TLS secret for the server:
```sh
kubectl create -n istio-system secret tls bookinfo-mtls-server \
  --key=bookinfo-server.key --cert=bookinfo-server.crt
```
Generate and sign the client certificate:
```sh
openssl req -newkey rsa:2048 -nodes -keyout bookinfo-client.key -out bookinfo-client.csr \
  -subj "/CN=bookinfo-client"

openssl x509 -req -in bookinfo-client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out bookinfo-client.crt -days 365
```
Create a secret for the CA certificate (used to validate client certs):
```sh
kubectl create -n istio-system secret generic bookinfo-mtls-server-cacert \
  --from-file=ca.crt=ca.crt
```
Note: Istio looks for a secret named bookinfo-mtls-server-cacert. Since the credentialName is bookinfo-mtls-server, the CA must be placed in bookinfo-mtls-server-cacert.

Restart the Istio ingress gateway to pick up the new secret:
```sh
kubectl rollout restart deployment istio-ingressgateway -n istio-system
```

#### Step 2: Update the Gateway to use MUTUAL TLS
Modify your Gateway resource like:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: bookinfo-mtls-server
      caCertificates: /etc/istio/ingressgateway-ca-certs/ca.crt
    hosts:
    - bookinfo.local.test
```
Apply the updated config:
```sh
kubectl apply -f gateway.yaml
```
You do not need to modify your VirtualService mTLS is enforced at the Gateway level.

#### Step 3: Test mTLS Access
Because browsers don't easily support client certs, we'll test using curl.

Access with client cert:
```sh
curl --cert bookinfo-client.crt --key bookinfo-client.key --cacert ca.crt \
  https://bookinfo.local.test:8443/productpage
```
You should get the product page successfully.

When using curl -v we can see the that we receive:
```
* Request completely sent off
< HTTP/2 200
```
Full output:
```
* Host bookinfo.local.test:8443 was resolved.
* IPv6: (none)
* IPv4: 127.0.0.1
*   Trying 127.0.0.1:8443...
* Connected to bookinfo.local.test (127.0.0.1) port 8443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: ca.crt
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Request CERT (13):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Certificate (11):
* (304) (OUT), TLS handshake, CERT verify (15):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=bookinfo.local.test
*  start date: Jun 29 01:41:11 2025 GMT
*  expire date: Jun 29 01:41:11 2026 GMT
*  common name: bookinfo.local.test (matched)
*  issuer: CN=bookinfo-ca
*  SSL certificate verify ok.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://bookinfo.local.test:8443/productpage
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: bookinfo.local.test:8443]
* [HTTP/2] [1] [:path: /productpage]
* [HTTP/2] [1] [user-agent: curl/8.7.1]
* [HTTP/2] [1] [accept: */*]
> GET /productpage HTTP/2
> Host: bookinfo.local.test:8443
> User-Agent: curl/8.7.1
> Accept: */*
>
* Request completely sent off
< HTTP/2 200
< server: istio-envoy
< date: Sun, 29 Jun 2025 02:07:12 GMT
< content-type: text/html; charset=utf-8
< content-length: 5290
< x-envoy-upstream-service-time: 744
<
<!DOCTYPE html>
<html>
...
</html>
* Connection #0 to host bookinfo.local.test left intact
```

Access without client cert:
```sh
curl -v https://bookinfo.local.test:8443/productpage
```
You should see an error like:
```
curl -v https://bookinfo.local.test:8443/productpage
* Host bookinfo.local.test:8443 was resolved.
* IPv6: (none)
* IPv4: 127.0.0.1
*   Trying 127.0.0.1:8443...
* Connected to bookinfo.local.test (127.0.0.1) port 8443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Request CERT (13):
* (304) (IN), TLS handshake, Certificate (11):
* SSL certificate problem: unable to get local issuer certificate
* Closing connection
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```
This confirms mTLS is working only clients with a valid certificate can connect.

---

## Clean-Up
Once you are done, clean up your local machine by running:
```sh
k3d cluster delete istio-demo
```

---

## Conclusion
In this post, we went beyond the basics and explored how to operate an Istio Gateway across several practical scenarios:
- Exposing services over HTTPS using SIMPLE TLS mode
- Simulating TLS misconfigurations and debugging gateway errors
- Serving multiple applications with different domains using multiple Gateways
- Configured mutual TLS to enforce secure, certificate-based authentication between clients and the Bookinfo service, ensuring both parties verify each other's identity over HTTPS.

These hands-on examples reinforce how powerful and flexible Istio Gateways can be when managing edge traffic in real-world environments. By understanding and practicing these patterns, you'll be better equipped to configure secure, scalable, and debuggable ingress for your service mesh.

---

## What's Next?
In the next post, we'll dive into routing inside the mesh using VirtualService: how to control traffic based on paths, shift traffic between versions, introduce delays or aborts, and build resilient communication patterns within your services.
Stay tuned! 