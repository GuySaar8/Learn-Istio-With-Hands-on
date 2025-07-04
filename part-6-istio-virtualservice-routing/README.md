# Part 6: Istio VirtualService Routing Inside the Mesh

![Istio VirtualService Routing](./image.png)

---

In the previous post, we explored how the Istio Gateway controls traffic entering the mesh through HTTPS and mTLS. But once traffic is inside the mesh, how does Istio decide where to send it? That's where VirtualService comes into play.
In this post, we'll dive deep into what a VirtualService is, how it fits into Istio's routing model, and how you can use it to control traffic within the mesh with precision. As always, we'll wrap up with hands-on examples.

---

## What Is an Istio VirtualService
In Istio, a VirtualService defines how requests are routed within the mesh after they arrive, either from the outside via an Ingress Gateway or from another service. While Kubernetes services offer basic DNS-based routing, Istio's VirtualService introduces advanced, application-aware.

You can think of a VirtualService as the routing contract between the consumer of a service and how that service is implemented and delivered. It doesn't replace a Kubernetes service it builds on top of it to give you fine-grained control over how requests reach your workloads.

A VirtualService allows you to define:
- Version-Aware Routing (Canary deployment, Blue-Green rollout, A/B testing)
- Routing rules based on paths, HTTP methods, headers, and other attributes
- Fault injection (delays, aborts) to test resilience
- Retry logic and timeouts
- Integration with Gateways for controlling ingress traffic

---

## Anatomy of a VirtualService
Here's a basic example of a VirtualService that routes all incoming traffic from bookinfo.local.test to the service  productpage at port 9080:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
  namespace: demo
spec:
  gateways:
    - istio-system/bookinfo-gateway
  hosts:
    - bookinfo.local.test
  http:
    - route:
      - destination:
          host: productpage.demo.svc.cluster.local
          port:
            number: 9080
```
Let's break this down:
- **gateways:** This field defines the scope of the VirtualService. If omitted, the routing rules apply only to internal mesh traffic (service-to-service communication within the mesh). When specified, it binds the VirtualService to one or more named Istio Gateways, enabling external traffic to be routed through them. This is essential for edge routing scenarios where services need to be exposed outside the mesh.

  For example:
  ```yaml
  gateways:
      - istio-system/bookinfo-gateway
    hosts:
      - bookinfo.local.test
  ```
  This tells Istio: "Apply this VirtualService only when requests come through the bookinfo-gateway and match the given host."
- **hosts:** The fully qualified domain name (FQDN) of the target service. This is required and can be a wildcard or specific service name. If you're integrating with a Gateway, this should match the host in the Gateway definition (bookinfo.local.test).
- **http:** This section contains one or more HTTP route rules. You can also define tcp or tls blocks for non-HTTP protocols.
- **route:** A list of destinations (one or more), each with an optional weight and a required host. This is where traffic gets split.
- **destination.host:** The service in the cluster to which traffic will be sent. This is typically the Kubernetes service name.

A single VirtualService can contain multiple rules, matches, and even layered fallbacks depending on routing logic. It is extremely flexible and designed to work in both simple and highly dynamic environments.

---

## Why VirtualService Matters
VirtualService is one of the most important tools in Istio for enabling advanced routing logic. It empowers platform teams and developers to define traffic behavior independently of application code or deployment tools. Here's a deeper look into its capabilities:

### 1. Version-Aware Routing
Route traffic between different subsets of a service using weighted distribution. This allows:
- Canary deployments 90% to v1, 10% to v2
- Blue/green rollouts
- A/B testing

### 2. Header, Path, and Method Matching
You can direct traffic based on request characteristics, including:
- URL path: /api/v2/*
- HTTP headers: user-agent, cookies, custom headers
- HTTP methods: GET, POST

This allows routing by user identity, API version, client type, and more.

### 3. Fault Injection
Simulate network faults or failures to test how your application handles problems:
- Add fixed delays: introduce 3 seconds of latency
- Return specific HTTP errors: 500 Internal Server Error
- Control the percentage of affected traffic

This is useful for chaos engineering, resiliency testing, and validating retry policies.

### 4. Retries and Timeouts
Define how Istio should handle failed requests:
- Automatic retries with optional retryOn rules
- Timeouts for slow responses
- Per-route policies to avoid cascading failures

For example, you could retry failed GET requests up to 3 times with a 2-second timeout per request.

---

## Hands-On: Configuring an Istio VirtualService
Now that we understand what a VirtualService is and how it helps us control traffic within the mesh, let's see it in action.
In this section, we'll walk through practical examples of how to use a VirtualService to implement some of its most powerful features such as version-aware routing, traffic matching, fault injection, timeouts, and retries. All of these examples will build on the same demo setup used in the previous posts.

![Istio Routing and Timing](./image2.png)

---

### Prepare
Before continuing with the hands-on examples, let's update the Gateway and the VirtualService configuration to match our current requirements.
In this scenario:
- We will not use HTTPS, so we'll revert the Gateway to use HTTP instead of TLS.
- We will not route directly to the web application, so a VirtualService is not needed at this point.

Update gateway.yaml file:
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
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.local.test"
---
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
  - match:
    - uri:
        prefix: /productpage
    route:
    - destination:
        host: productpage.demo.svc.cluster.local
        port:
          number: 9080
```
Apply it:
```sh
kubectl apply -f gateway.yaml
```

---

### Scenario 1: Routing Bookinfo Microservices by Path
The Bookinfo application includes several microservices: productpage, details, reviews, and ratings. When a user opens /productpage, that page makes internal calls to the other services. Each service is responsible for rendering part of the page.

In this scenario, we'll configure a single VirtualService that:
- Matches traffic by URL path: /productpage, /details
- Routes requests to the appropriate microservice inside the mesh
- Uses the same external hostname (bookinfo.local.test) for all paths

This approach mimics a real-world frontend/backend structure, where multiple APIs or microservices are routed based on the path prefix.

#### Step 1: Define the VirtualService
Let's update our VirtualService to match the routing for all the microservices of the Bookinfo application.

gateway.yaml:
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
    - match:
        - uri:
            prefix: /productpage
      route:
        - destination:
            host: productpage.demo.svc.cluster.local
            port:
              number: 9080
    - match:
        - uri:
            prefix: /details
      route:
        - destination:
            host: details.demo.svc.cluster.local
            port:
              number: 9080
    - match:
        - uri:
            prefix: /ratings
      route:
        - destination:
            host: ratings.demo.svc.cluster.local
            port:
              number: 9080
    - match:
        - uri:
            prefix: /reviews
      route:
        - destination:
            host: reviews.demo.svc.cluster.local
            port:
              number: 9080
```
Apply it:
```sh
kubectl apply -f gateway.yaml
```

#### Step 2: Test It
Via the browser go to the following URL pages
- http://bookinfo.local.test:8080/productpage
- http://bookinfo.local.test:8080/details/123
- http://bookinfo.local.test:8080/ratings/123
- http://bookinfo.local.test:8080/reviews/123

Notice how the same base URL, combined with different path suffixes, allows access to individual microservices depending on how the VirtualService is configured.

**Why This Matters**
This example highlights path-based routing, which is one of the most common use cases in service mesh environments. Instead of exposing every microservice through its own domain or Gateway, you can consolidate all routes under a single hostname and control routing using clean path prefixes.

This gives you:
- Centralized traffic management
- A clean and stable public interface
- The flexibility to add advanced behaviors later (e.g. retries, mirroring, or canary releases)

---

### Scenario 2: Version-Aware Routing (Canary Release)
Istio's VirtualService allows you to split traffic across multiple destinations. This is the foundation for canary releases, A/B testing, and gradual rollouts.

In this simplified example of Canary deployment, we'll simulate "the upgrading process" of our existing web application (a basic NGINX deployment) to the productpage service from the Bookinfo application in a controlled and managed way.

Instead of switching everything at once, we'll split incoming traffic gradually between the two, letting us observe real behavior under partial load.

In real-world canary deployments, you'd use tools such as Argo Rollouts, Flagger, or route between two versions of the same service like v1 and v2 of reviews. That requires defining subsets with a DestinationRule—something we'll cover in the next blog post.

#### Step 1: Update the VirtualService
We'll direct 90% of traffic to web, and 10% to bookinfo, when visiting /productpage.

gateway.yaml:
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
    - match:
        - uri:
            prefix: /productpage
      rewrite:
        uri: "/"   # this removes /productpage from the path
      route:
        - destination:
            host: web.demo.svc.cluster.local
            port:
              number: 80
          weight: 100
        - destination:
            host: productpage.demo.svc.cluster.local
            port:
              number: 9080
          weight: 0
```
Apply it:
```sh
kubectl apply -f gateway.yaml
```

**Why We Needed to Add a Rewrite**
In our application setup, we wanted to route requests coming to /productpage to an NGINX service running in our Kubernetes cluster. However, the NGINX service itself was not configured to handle paths prefixed with /productpage. 
To resolve this, we used Istio's rewrite feature within the VirtualService configuration. The rewrite rule allows us to modify the incoming request URL before it reaches the destination service. By rewriting the URI from /productpage/... to just /..., we ensured that the backend service receives the path in the format it expects.

This small but crucial change allows us to decouple the external routing structure from the internal service expectations, keeping the configuration clean and the services unaware of gateway-level routing decisions.

#### Step 2: Test the Rollout Behavior
Visit the app in your browser:
http://bookinfo.local.test:8080/productpage
You'll mostly land on the web page, but sometimes be routed to bookinfo's product page depending on the traffic split. This is controlled entirely by Istio's routing logic.
Continue incrementally shifting traffic from web to bookinfo by adjusting the weights, until all traffic (100%) is directed to bookinfo, finalizing the Canary deployment.

---

### Scenario 3: Header-Based Routing (Mobile vs. Desktop)
Istio's VirtualService allows you to route traffic based on HTTP headers, URL paths, and request methods. This gives you full control over how different client types, mobile vs. desktop, API clients vs. browsers, are handled by your services.

In this scenario, we'll demonstrate how to route mobile users identified by their User-Agent HTTP header to the Bookinfo productpage service, while all other traffic continues going to our existing web service.

This kind of routing can be useful for:
- Serving optimized views to mobile clients
- A/B testing with specific client types
- Safely deploying new behavior for a subset of users

Note: This example only matches the User-Agent header. In a real production setup, you could match on cookies, custom headers, paths, or even HTTP methods to drive more granular routing decisions.

#### Step 1: Update the VirtualService to Match on Headers
We'll define two routing rules:
- If the request comes from a mobile user-agent, send it to productpage.
- All other traffic goes to web.

gateway.yaml:
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
    - match:
        - uri:
            prefix: /productpage
          headers:
            user-agent:
              regex: ".*Mobile.*"
      route:
        - destination:
            host: productpage.demo.svc.cluster.local
            port:
              number: 9080
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

#### Step 2: Test the Header-Based Routing
To test routing behavior based on headers, we'll use curl with the -A flag (short for --user-agent) to simulate a browser or mobile device.

Run the following command:
```sh
curl -A "Mobile Safari" http://bookinfo.local.test:8080/productpage
```
This tells Istio: "The client is on a mobile browser", because the User-Agent contains Mobile, so traffic should route to productpage.

Now try it without a mobile user agent:
```sh
curl -A "Mozilla Firefox" http://bookinfo.local.test:8080/productpage
```
This time, traffic should route to the web service.

**Question: What service will this URL reach?**
```sh
curl -A "Mobile Safari" http://bookinfo.local.test:8080
```
**Answer:** If you said the web service, you were correct.

Why? Although the request includes the User-Agent header that matches the "mobile" criteria, it does not match the specific uri path defined in the route /productpage. Because both conditions were required for a match, the route didn't apply. As a result, the request fell through to the default route, which sends traffic to the web service.

This example shows how VirtualService lets you tailor user experiences based on request metadata all without touching your app code or deployments.

---

### Scenario 4: Fault Injection (Simulating Latency)
One of the powerful features of Istio's VirtualService is its ability to inject faults into live traffic. This is useful for:
- Testing how applications behave under failure conditions
- Validating retry and timeout policies
- Practicing chaos engineering
- Ensuring that frontends degrade gracefully when dependencies are slow

In this scenario, we'll simulate artificial latency in the bookinfo service. When users access the /productpage, the response will be intentionally delayed by 3 seconds.

This mimics what would happen if a downstream dependency (like a database or third-party API) was slow allowing you to verify whether your frontend stays responsive, displays loading states, or fails completely.

#### Step 1: Inject a Fixed Delay Using VirtualService
We'll define a VirtualService that matches requests to /productpage and delays them. In addition to that requests that matches /details will be aboretd 50% of the time.

gateway.yaml:
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
    - match:
        - uri:
            prefix: /productpage
      fault:
        delay:
          fixedDelay: 3s        # Always delay by 3 seconds
          percentage:
            value: 100          # Apply to 100% of requests
      route:
        - destination:
            host: productpage.demo.svc.cluster.local
            port:
              number: 9080
    - match:
        - uri:
            prefix: /details
      fault:
        abort:
          percentage:
            value: 50           # 50% of incomming traffic
          httpStatus: 400       # return status code 400
      route:
        - destination:
            host: details.demo.svc.cluster.local
            port:
              number: 9080
```
Apply it:
```sh
kubectl apply -f gateway.yaml
```

#### Step 2: Observe the Delay
Now test it from your terminal using curl and measure the total time:
```sh
time curl -k http://bookinfo.local.test:8080/productpage
```
You should notice that the response takes at least 3 seconds. Here's what a successful output looks like:
```
real 0m3.144s
user 0m0.007s
sys 0m0.010s
```
This confirms that Istio injected the delay as configured.

#### Step 3: Observe the Abort
Since we configured Istio to abort 50% of requests with an HTTP 400 status code:
- About half the requests will fail with a 400 Bad Request response.
- The other half will succeed and return the normal response from the details service.

You can try running the request multiple times to observe both outcomes. For example:
```sh
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" -k http://bookinfo.local.test:8080/details/123; done
```
This will print the HTTP status code for each request. You should see a mix of 200 and 400 responses.

**Why Fault Injection Matters**
Injecting artificial faults like delays or aborts is a powerful technique for testing how your system behaves under stress before real-world issues strike.

Here's why this matters:
- Validates frontend behavior: Can your UI gracefully handle slow responses? Are there proper loading indicators, retry prompts, or timeout fallbacks?
- Reveals performance regressions early: Simulating latency helps catch sluggish code paths or overloaded services before they affect users.
- Strengthens resilience testing: You can combine delay or abort faults with retry logic to verify that retry settings are actually helping not hurting your availability.

**Tip for Realistic Testing in Staging**
In a staging environment, consider reducing the fault.delay.percentage.value from 100% to a small number like 1–5%. This introduces random latency on a subset of requests while keeping the environment stable enough for other tests.

Why bother doing this in staging?
- It simulates real-world conditions like slow clients, intermittent packet loss, or backend hiccups.
- It helps catch missing timeout settings, retry loops, or poor user experience that would otherwise be masked in a "perfect" test environment.

By introducing a bit of chaos, you build confidence that your system can withstand it.

---

### Scenario 5: Retries and Timeouts
In this scenario, we'll configure Istio to automatically retry failed requests and limit how long it waits for a response. This is crucial for building resilient applications that can tolerate intermittent issues like transient network failures or overloaded services.

Retries and timeouts are your first line of defense against unreliable networks or flaky services:
- Retries help recover from temporary failures without user-visible errors.
- Timeouts prevent your system from waiting too long and blocking upstream components.

Together, they help keep your system responsive and fault-tolerant.

#### Step 1: Add Retry and Timeout Settings
Update the existing VirtualService for the productpage service. Create or edit a file called gateway.yaml:
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
  - match:
    - uri:
        prefix: /productpage
    route:
    - destination:
        host: productpage.demo.svc.cluster.local
        port:
          number: 9080
    retries:
      attempts: 1000
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
    timeout: 20s
```
This configuration tells Istio:
- Istio will retry failed requests up to 1000 times.
- Each retry attempt has a timeout of 2 seconds.
- The overall request will be aborted after 20 seconds total.
- Only retry on:
  - gateway-error: Upstream returned a 502, 503, or 504
  - connect-failure: Connection could not be established
  - refused-stream: HTTP/2 stream was refused

Apply it:
```sh
kubectl apply -f gateway.yaml
```

#### Step 2: Simulate Failures and Observe Behavior
Now we'll test Istio's retry behavior by simulating a failure in the productpage service.

Patch the svc so the traffic will not reach the productpage pod
```sh
kubectl patch svc productpage -n demo - type='json' -p='[{"op": "replace", "path": "/spec/ports/0/targetPort", "value": 80}]'
```
2. Test with curl (expect a delay due to retries):
```sh
time curl -k http://bookinfo.local.test:8080/productpage
```
3. Expected output and explantion
Because the service was misconfigured, Istio's Envoy proxy couldn't connect to the backend on the expected port, resulting in a connect-failure.
Thanks to our retry policy, Istio continuously retried the request, each attempt waiting up to 2 seconds before failing.
The 1000 attempted retreis took longer then the timeout of 20 seconds before finally returning an error.
When we ran:
```sh
time curl -k http://bookinfo.local.test:8080/productpag
```
The output was:
```
upstream request timeout
real    0m20.028s
user    0m0.006s
sys     0m0.006s
```
4. Revert the change of the productpage svc
```sh
kubectl patch svc productpage -n demo --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/targetPort", "value": 9080}]'
```

**Why Retries and Timeouts Matter**
Retries and timeouts are critical tools for building resilient, user-friendly services in distributed systems.
This example shows how Istio can buffer and absorb transient or even prolonged service failures, giving your system time to recover or fallback gracefully, instead of failing immediately.
Together, retries and timeouts make your applications more fault-tolerant, faster to fail when needed, and better equipped to operate reliably in real-world production environments.

---

## Clean-Up
Once you are done, clean up your local machine by running:
```sh
k3d cluster delete istio-demo
```

---

## Conclusion
These hands-on scenarios show how powerful Istio VirtualService can be when routing traffic inside the mesh. From canary deployments to chaos testing and beyond, VirtualService gives you fine-grained control over traffic behavior all declaratively, without touching your app code.

---

## What's Next?
In the final post of this 7-part Istio blog series, we'll dive into DestinationRule, PeerAuthentication, and Advanced Traffic Policies. You'll learn how DestinationRule complements VirtualService by managing connection-level settings at the destination such as TLS modes, connection pools, and load balancing strategies while PeerAuthentication handles security at the workload level.
This last installment will bring together everything we've covered so far and deepen your understanding of how to fine-tune and secure traffic within your service mesh.
Stay tuned! 