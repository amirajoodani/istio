# istio
Service meshes are a vital component of a company’s infrastructure. Istio is an open source project and the leading service mesh implementation that is gaining rapid adoption. To understand why service meshes are so important and why they are becoming increasingly common, we must look back at the shift from monoliths to microservices and cloud native applications, and understand the problems that stemmed from this shift.<br>

We must also review other technologies that were developed as a response to these problems, and try to understand why these other solutions, in many ways, fall short of their objective.<br>

We will then be in a position to review the architecture of Istio to understand both how and why service meshes elegantly address these problems.<br>

Service meshes elegantly solve problems in the areas of security, observability, high availability, and scalability. They enable enterprises to respond to situations quickly without requiring development teams to modify and redeploy their applications to effect a policy change. A service mesh serves as the foundation for running cloud native applications.<br>

# New Problems
Microservices brought several benefits. Organizations were able to organize into smaller teams. Codebases didn't all have to be written in the same language. Individual codebases shrank and consequently became simpler and easier to maintain and deploy. The deployment of a single microservice entailed less risk. Continuous integration got easier. There now existed contracts in the form of APIs for accessing other services, and developers had fewer dependencies to contend with.<br>

But the new architecture also brought with it a new set of challenges. Certain operations that used to be simple became difficult. What used to be a method call away became a call over the network. The address of that target service in an increasingly dynamic environment became difficult to resolve. How does one service know if another is available?<br>

# Challenges posed by the new architecture:
<b>Service discovery</b> : How does a service discover the network address of other services? <br>
<b>Load balancing </b>: Given that each service is scaled horizontally, the problem of load-balancing was no longer an ingress-only problem.<br>
<b>Service call handling</b> :How to deal with situations where calls to other services fail or take an inordinately long time? An increasing portion of developers' codebases had to be dedicated to handling the failures and dealing with long response times, by sprinkling in retries and network timeouts.<br>
<b>Resilience</b> : Developers had to learn (perhaps the hard way) to build distributed applications that are resilient and that prevent cascading failures.<br>
<b>Security</b> : How do we secure our systems given this new architecture has a much larger attack surface? Can a service trust calls from other services? How does a service identify its caller?<br>
<b>Diagnosis and troubleshooting </b>: Stack traces no longer provided complete context for diagnosing an issue. Logs were now distributed. How does a developer diagnose issues that span multiple microservices?<br>
<b>Resource utilization</b> :Managing resource utilization efficiently became a challenge, given the larger deployment footprint of a system made up of numerous smaller services.<br>
<b>Automated testing </b>: End-to-end testing became more difficult.<br>
<b>Traffic management </b>:The ability to route requests flexibly to different services under different conditions started becoming a necessity.<br>

# Service Meshes
Engineers struggled with similar problems at Lyft, moving away from monoliths toward a cloud native architecture.

At Lyft, Matt Klein and others proposed a different approach to solving these same problems: to bundle infrastructural capabilities completely out of process, in a manner separate from the main running application. Essentially every service in a distributed system would be accompanied by its own dedicated proxy running out-of-process.

By routing requests in and out of a given application through its proxy, the proxy would have the opportunity to perform services on behalf of its application in a transparent fashion. This proxy could be made to retry failed requests, it could be configured with specific network timeouts, and circuit-breaking logic. The proxy could also be made to encapsulate the logic of performing client-side load balancing requests to other services.

The list doesn't stop there. The proxy could act as a security gateway, also known as a Policy Enforcement Point. Connections can be upgraded from plain HTTP to encrypted traffic with mutual TLS. The proxy could be made to collect metrics such as request counts, durations, response codes and more, and expose those metrics to a monitoring system, thereby removing the burden of managing metrics collection and publishing from the development teams.

The application of this proxy at Lyft helped solve many of the problems that its development teams were running into, and helped make their migration away from monoliths a success. Matt Klein subsequently open sourced the project and named it Envoy.

At the same time, the advent of containerization (Docker) and container orchestrators (Kubernetes) began addressing problems of resource utilization and freed operators from the mundane task of determining where to run workloads.

Kubernetes Pods provided an important intermediary construct that, on the one hand, allowed for isolation within the container but on the other, for multiple loosely coupled containers to be bundled together as a single unit.

Kubernetes came from Google, and Google was looking to build this same out-of-process infrastructural capability on top of Kubernetes. The Istio project started at Google, and as it turns out, Istio saw in Envoy the perfect building block for its service mesh. The Istio control plane would automate the configuration and synchronization of proxies deployed onto Kubernetes as sidecars inside each Pod.

The Kubernetes API server's capabilities could be leveraged to automate service discovery and communicate the locations of service endpoints directly to each proxy. Fine-grained configuration of proxies could be performed by exposing Kubernetes Custom Resource Definitions (CRDs).

The Istio project was born. <br>
<img width="866" height="571" alt="1" src="https://github.com/user-attachments/assets/cbd414f0-99e6-42b2-8c9b-196a8f3b4d7e" /> <br>

# Istio Architecture
As shown in the illustration in the previous section, the basic idea behind Istio is to push microservices concerns into the infrastructure by leveraging Kubernetes. This is implemented by bundling the Envoy proxy as a sidecar container directly inside every Pod.

Note: In the Advanced Topics chapter, we show how a service mesh can be extended to include workloads running on VMs, outside Kubernetes.

In terms of implementation, Istio's main concerns are, therefore, solving the following problems:

Ensuring that each time a workload is deployed, an Envoy sidecar is deployed alongside it.
Ensuring traffic into and out of the application is transparently diverted through the proxy.
Assigning each workload a cryptographic identity as the basis for a more secure computing environment.
Configuring the proxies with all the information they need to handle incoming and outgoing traffic.
We will explore each of these concerns more in-depth in the following sections.<br>

# Sidecar Injection
Modifying Kubernetes deployment manifests to bundle proxies as sidecars with each pod is both a burden to development teams, error-prone and not maintainable.

Part of Istio's codebase is dedicated to providing the capability to automatically modify Kubernetes deployment manifests to include sidecars with each pod.

This capability is exposed in two ways, the first and simpler mechanism is known as manual sidecar injection, and the second is called automatic sidecar injection. <br>

# Manual Sidecar Injection
Istio has a command-line interface (CLI) named istioctl with the subcommand kube-inject. The subcommand processes the original deployment manifest to produce a modified manifest with the sidecar container specification added to the pod (or pod template) specification. The modified output can then be applied to a Kubernetes cluster with the kubectl apply -f command.

With manual injection, the process of altering the manifests is explicit.<br>

# Automatic Sidecar Injection
With automatic injection, the bundling of the sidecar is made transparent to the user.

This process relies on a Kubernetes feature known as Mutating Admission Webhooks, a mechanism that allows for the registration of a webhook that can intercept the application of a deployment manifest and mutate it before the final, modified specification is applied to the Kubernetes cluster.

The webhook is triggered according to a simple convention, where the application of the label istio-injection=enabled to a Kubernetes namespace governs whether the webhook should modify any deployment or pod resource applied to that namespace to include the sidecar.<br>



