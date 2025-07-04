# Part 2: Getting Started with Istio: The Backbone of Your Service Mesh

![Istio Control Plane Cloud](./image.png)

---

In the previous blog post, we explored **what a service mesh is** and why it matters in modern microservices architecture. Now it's time to dig deeper into one of the most widely used service mesh implementations: **Istio**.

# What is Istio?

Istio is an **open-source service mesh** that provides a powerful way to connect, secure, control, and observe microservices all without requiring changes to your application code.

With Istio, you get:

* **Traffic management** (routing, retries, load balancing)
* **Security** (mTLS, authentication, and authorization)
* **Observability** (metrics, tracing, logs)
* **Policy enforcement** (access control, rate limiting)

Rather than baking these features into each service, Istio provides them at the platform level letting you manage networking, security, and telemetry centrally.

# Istio Components

Istio is made up of two primary layers:

**Control Plane (istiod):**  
Manages the overall behavior of the mesh. It handles service discovery, distributes configuration and policies, manages certificates for secure communication, and pushes updates to the proxies using the xDS protocol.

**Data Plane:** 
Responsible for handling all network traffic between services. It includes:

* **Istio Ingress Gateway:** A standalone Envoy proxy deployed at the edge of the mesh to manage incoming traffic from external clients, including TLS termination and routing based on Gateway and VirtualService configurations.
* **Envoy sidecar proxies:** Injected into each pod to intercept and manage internal service-to-service traffic, applying routing rules, security policies, retries, and telemetry collection.


# The Control Plane `istiod` The Brain Behind Istio

![Istio Mesh Brain](./image2.png)

`istiod` is the **central control plane** of Istio. Introduced in **Istio v1.5**, it unified several older components into a single binary:

* **Pilot:** Handles service discovery and traffic management.
* **Citadel:** Manages certificate issuance and mutual TLS (mTLS).
* **Galley:** Performs configuration validation.

By merging these into `istiod`, Istio simplified deployment and improved performance reducing the number of running components and centralizing configuration and coordination.

## What Does `istiod` Do?

`istiod` **doesn't handle actual user traffic. Instead, it configures and coordinates the sidecar proxies that do.** Here's what it handles:

**_Service Discovery_**

Watches Kubernetes (or other platforms) for new services and workloads, and distributes this information to Envoy sidecars.

**_Configuration Distribution_**

Takes Istio CRDs like `VirtualService`, `Gateway`, and `DestinationRule`, converts them into Envoy config using the **xDS API**, and pushes updates to proxies in real time.

**_Certificate Authority (CA)_**

Issues and rotates certificates automatically. Istio uses **mutual TLS (mTLS)** for secure, encrypted communication between services no app changes required.

**_Sidecar Injection_**

Automatically injects the Envoy proxy sidecar into your pods if auto-injection is enabled for a namespace or deployment.

## Service Discovery - How `istiod` Finds and Shares Services

`istiod` integrates deeply with Kubernetes to keep the service mesh aware of everything happening in the cluster. It does this by watching the Kubernetes API server using standard informers a Kubernetes mechanism for subscribing to resource changes. These informers track events such as updates to services, pods, or endpoints, and notify `istiod` in real time.

The key resources `istiod` watches include:

* **Services** define the available service names, ports, and selectors that map to backend pods
* **Endpoints** and **EndpointSlices** track the actual pod IPs behind each service, updated dynamically as pods come and go
* **Pods** and **Deployments** provide details about running workloads, including labels and metadata used in service matching and policy enforcement
* **Istio CRDs** such as `VirtualService`, `DestinationRule`, etc., which define traffic rules, failover behavior, and mesh-specific policies

## How it Works Behind the Scenes

When something in the cluster changes like a new pod starting, a service scaling, or a config update:

1. **Kubernetes emits an event**, notifying watchers of the change.
2. `**istiod**` **detects the update** and refreshes its **internal model** of the mesh, which includes:
* Service-to-pod mappings
* Routing and traffic rules
* Security and policy settings

3. `**istiod**` **uses the xDS API** (Envoy's dynamic configuration protocol) to push updated configuration to all relevant **Envoy sidecar proxies**.

This includes:

* What services are available (CDS Cluster Discovery Service)
* Which pods are behind them (EDS Endpoint Discovery Service)
* How to route requests (RDS Route Discovery Service)
* How traffic should be handled (LDS Listener Discovery Service)

# The Data Plane Istio Ingress Gateway and The Sidecar Proxy Model

![Istio Control Plane Brain](./image3.png)

# Istio Ingress Gateway

The **Istio Ingress Gateway** is an **Envoy proxy** that sits at the **edge of your cluster**. It handles **incoming traffic from outside the mesh** and routes it to internal services based on the configuration defined in your Istio resources.

Key responsibilities include:

* **Accepting external HTTP/TCP traffic** at the cluster boundary
* **Terminating TLS** (if configured), or passing it through
* **Routing traffic** based on Istio `Gateway` and `VirtualService` configuration
* **Applying mesh policies** like retries, timeouts, or headers at the edge

While sidecars manage traffic **_inside_** the mesh by `**istiod**`, **Istio Ingress Gateway** manages traffic coming **_into_** the mesh. Think of it as your app's front door with smart locks and routing rules.

# The Sidecar Proxy Model

Istio uses a **sidecar pattern**: for every pod, it injects a lightweight **Envoy proxy** container alongside your application container.

This proxy:

* Intercepts all inbound and outbound traffic
* Applies security policies, telemetry, retries, circuit breakers
* Gets real-time updates from `istiod`

By offloading all this logic to the proxy, your services can stay focused on business logic.

# Summary

## Istiod: The Control Plane Brain

* Continuously watches the Kubernetes API for changes to services, pods, endpoints, and Istio CRDs
* Builds and maintains an internal model of the mesh, including service-to-pod mappings and traffic rules
* Uses the xDS API to push configuration to all Envoy proxies in the mesh
* Enables real-time updates to routing, security, and policy enforcement without redeploying applications

## Istio Ingress Gateway: The Mesh Entry Point

* A dedicated Envoy proxy deployed at the edge of the cluster
* Handles all incoming external traffic into the mesh
* Terminates TLS (if configured) or passes it through
* Routes traffic to internal services based on Gateway and VirtualService resources
* Applies policies like retries, timeouts, and headers at the boundary

## Envoy Sidecars: The Traffic Enforcers Inside the Mesh

* Injected automatically into every pod alongside your application container
* Intercept and manage all inbound and outbound traffic for their pod
* Apply routing rules, security policies, retries, and telemetry collection
* Receive real-time updates from istiod to reflect the current mesh state
* Ensure consistent traffic behavior across all services without modifying application code

This separation of concerns allows Istio to dynamically control traffic behavior without redeploying your applications, providing both flexibility and security across your entire service mesh.

# What's Next?

Now that we understand Istio's architecture istiod, the Ingress Gateway, and the sidecar proxies it's time to look deeper into how these components interact within the service mesh. In the next article, we'll explore:

* A clear and visual breakdown of the **Control Plane vs. Data Plane** what each is responsible for and how they communicate.
* A quick overview of **how sidecar injection works** and why it's mostly invisible but critical.
* Some key things that **istiod does not do** and why that matters when planning your architecture.

This deeper dive will help you reason about Istio's design and prepare you to confidently navigate its traffic management features.

Stay tuned!

---

[Original post on Medium](https://medium.com/israeli-tech-radar/getting-started-with-istio-the-backbone-of-your-service-mesh-20ea01e3553c) 