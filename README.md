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

# Routing Application Traffic Through the Sidecar
With the sidecar deployed, the next problem is ensuring that the proxy transparently captures the traffic. The outbound traffic should be diverted from its original destination to the proxy, and inbound traffic should arrive at the proxy before the application has a chance to handle the incoming request.

This is performed by applying iptables rules . Routing Application Traffic Through the Sidecar
With the sidecar deployed, the next problem is ensuring that the proxy transparently captures the traffic. The outbound traffic should be diverted from its original destination to the proxy, and inbound traffic should arrive at the proxy before the application has a chance to handle the incoming request.

This is performed by applying iptables rules. The video by Matt Turner titled Life of a Packet through Istio explains elegantly how this process works.

In addition to the Envoy sidecar, the sidecar injection process injects a Kubernetes init container. This init-container is a process that applies these iptables rules before the Pod containers are started.

Today Istio provides two alternative mechanisms for configuring a Pod to allow Envoy to intercept requests. The first is the original iptables method, and the second uses a Kubernetes CNI plugin. <br>

# Assigning Workloads an Identity
The basis for a secure mesh is strong identity. We often associate the concept of identity with an end user. But services also bear identity. For example, when shopping on barnesandnoble.com, the server offers your browser a certificate that allows it to assert the server's identity.

In Istio, each workload is assigned an X.509 cryptographic identity that adheres to the SPIFFE (Secure Production Identity Framework for Everyone) framework.

Based on the SPIFFE framework, Istio encodes a SPIFFE ID into a service's certificate. The SPIFFE ID is a URL in the form spiffe://<trust domain>/<workload identifier>.

In Istio, the trust domain value is typically drawn from the Kubernetes cluster's domain, while the workload identity is a combination of the service's namespace and service account fields.

Inside each sidecar, an Istio agent bootstraps Envoy and the service identity by sending a certificate signing request (CSR) to Istio, and making the resulting signed certificate accessible to Envoy securely (via its xDS API).

This course dedicates an entire chapter to Istio security.<br>

# Configuring Envoy
When an application makes a call to another service, that call is now intercepted by its sidecar. But how does Envoy know how to route that request? In the other direction, when a request arrives from another service at a sidecar, how does Envoy know whether that request should be allowed through?

The job of configuring the proxies with all the information they need to handle both incoming and outgoing traffic falls to the Istio control plane.

It is important to point out that only Envoy is in the path of live requests and responses between services; the Istio control plane is not.

Let us explore a number of simple scenarios in order to better understand the kind of configuration that the sidecars require.

Imagine a sidecar for an instance of service A intercepting an outgoing request to service B. Service B may be backed by a Kubernetes Deployment with, say, three replicas. Service A's sidecar must know about all three endpoints: their network address, whether they're healthy, optionally the desired load balancing strategy when calling service B, and optionally other network configuration parameters such as request timeouts, number of retries, outlier detection, and more.
Imagine an operator specifying that all mesh traffic should be encrypted. The sidecar needs to be aware of this configuration in order to decide whether or not to upgrade the connection.
Imagine a situation where we're doing A/B testing. We have two subsets of service B's endpoints, with a rule to send certain types of requests to one subset and the rest to the other. Those subsets and rules must be communicated to the Envoy sidecar in order for it to adhere to this policy.
Imagine a service mesh with over a hundred microservices, and your job is to configure each sidecar manually. That proposition is untenable. This, in a nutshell, is the job that Istio performs. One could say that Istio automates the configuration of all sidecars in the mesh to do their job of routing traffic according to a defined network policy, security policy, routing policy, and so on.

One point to appreciate is that the configuration of sidecars is not a one-time, static operation. It's a dynamic reconciliation process, because Kubernetes is a dynamic environment.

Imagine a scenario where a deployment is auto-scaled from two to three replicas. Information about the newly-created service endpoint must be communicated to all the sidecars in the mesh, so that the new endpoint can join the pool of load balancing endpoints that can be reached from other services.

This brings us to a final and important point about Envoy: Envoy has the ability to receive configuration updates via API and to reload its configuration "live", without requiring a restart. This API is known as Envoy's discovery API, often abbreviated xDS.

Istio is then the control plane that continuously pushes configuration updates to sidecars each time the mix or number of services in the mesh changes, or each time we update policies that affect the mesh. <br>

# Envoy at the Edge
No service mesh is an island.

Envoy, after all, is a proxy, and Istio leverages Envoy not only for proxying requests within the mesh, but also as the mechanism for handling ingress and egress traffic (i.e., traffic coming from a source outside the mesh, or traffic bound to a destination outside the mesh).

Indeed, there exist today multiple open-source and commercial implementations of Kubernetes Ingress controllers built on Envoy. Contour and Emissary-ingress are two examples.

Aside:  The Envoy project announced the Envoy Gateway project, a collaborative effort to develop an open source solution for Ingress based on Envoy. 

Two additional important components of the Istio architecture are Istio's Ingress Gateway and its Egress Gateway. Both are based on Envoy. They support the original Kubernetes Ingress CRD. However, Istio provides its own Gateway Custom Resource Definition (CRD) for configuring ingress and egress more flexibly.

We delve into these topics in more detail in the chapter on traffic management.

The illustration below captures all of the Istio components, including the edge gateways.<br>
<img width="1254" height="610" alt="2" src="https://github.com/user-attachments/assets/389c11a0-ec0b-4aa9-ab26-1a8eb3672c1a" /> <br>

# istio Installation
Installation Configuration Profiles
Istio service mesh has numerous configuration settings that operators can update before installing Istio. To group the most common configuration settings into a higher-level abstraction, Istio uses the concept of configuration profiles.

The configuration profiles contain different configuration settings for the control plane as well as the data plane of Istio. The installation configuration profiles are expressed through the Istio Operator API and the IstioOperator resource.

Six configuration profiles are currently available, as shown in the list below. To get an up-to-date list of Istio configuration profiles, run the istioctl profile list command. <br>
<b>Default profile</b> : The default profile is meant for production deployments and deployments of primary clusters in multi-cluster scenarios. It deploys the control plane and ingress gateway.<br>
<b>Demo profile</b> : The demo profile is intended for demonstration deployments. It deploys the control plane and ingress and egress gateways and has a high level of tracing and access logging enabled.<br>
<b>Minimal profile</b> : The minimal profile is equivalent to the default profile but without the ingress gateway. It deploys the control plane.<br>
<b>External profile </b>: The external profile is used for configuring remote clusters in a multi-cluster scenario. It does not deploy any components.<br>
<b>Empty profile </b>: The empty profile is used as a base for custom configuration. It does not deploy any components.</br>
<b>Preview profile </b>: The preview profile contains experimental features. It deploys the control plane and ingress gateway.</br>
<b>Ambient profile </b>: The ambient profile is designed to help you get started with ambient mesh. Ambient mode is currently in the Alpha phase (as of April 2024). Please do not use ambient mode in production yet.</b>
To install Istio using the Istio CLI, we can use the --set flag and specify the profile like this:

istioctl install --set profile=demo

Later, we will cover how to install and customize Istio by creating an IstioOperator resource and installing it using the Istio CLI. We will also cover how to use Helm and deploy the Istio Helm charts.</br>

# Using Istio Operator API
The Istio Operator API and the IstioOperator resource allow us to install and configure Istio on a Kubernetes cluster. At a high level, we can separate the configuration in the IstioOperator resource into the following sections:

Global
The global section allows us to configure the profile name, root Docker image path, image tags, namespace, revision, and so on.
Mesh configuration (meshConfig)
The meshConfig section includes the configuration of the control plane components. For example, in this section, we can configure access log format, log encoding, set up default proxy configuration, discovery selectors, trust domains, and more.
Component configuration (components)
The components section allows us to enable or disable individual components, install additional components (multiple ingress or egress gateways, for example), and configure Kubernetes resource settings for individual components. For example, for each component (e.g., pilot, ingress, or egress gateways), we can configure the CPU and memory requests and limits, annotations, labels, replica counts, and other settings in the Kubernetes resources.
Within the IstioOperator resource, we specify the desired state of Istio components. We can apply or deploy the resource to the Kubernetes cluster using the Istio CLI and the Code in a paragraph or file content: install command.

Once we have created the IstioOperator resource, we can install it on the cluster using the install command:

istioctl install -f my-operator-resource.yaml

# Using Helm
Helm is a Kubernetes package manager that helps install and upgrade complex applications on Kubernetes. A fundamental building block of Helm is a Helm Chart, a collection of YAML manifests.

When using Helm, there are three different Helm charts we need to be aware of, listed in the order we would install them:

Base chart (istio/base)
The base chart includes cluster-wide resources such as the validating webhook configuration resource, service accounts, cluster roles and bindings, and other resources to ensure backward compatibility.
Istiod chart (istio/istiod)
The istiod chart contains Istio’s control plane installation. It includes the istiod deployment and service, mutating webhook configuration (facilitates automatic sidecar injection into deployments), and other resources for the control plane.
Gateway chart (istio/gateway)
The gateway chart is used for deploying ingress and egress gateways to the cluster. It includes the service and deployment resources and other supporting resources.
Before installing the charts, we need to manually create the root namespace (i.e., istio-system) and use the helm install command to install the individual charts. Typically, we install the base and istiod charts to the istio-system namespace and gateway charts into separate namespaces.

Here is how we could install the istiod chart, for example:

helm install istiod istio/istiod -n istio-system

The first parameter in the above command is the release name, followed by the chart name.

To check on the installation progress, we can pass the release name (e.g. istiod) to the status command:

helm status istiod -n istio-system

# Using Helm: Updating the Configuration
We can provide custom configuration settings to individual Helm charts at installation time. To review settings that can be updated, we can use the show values command like this:

helm show values istio/istiod
```bash
#.Values.pilot for discovery and mesh wide config 

## Discovery Settings
pilot:
  autoscaleEnabled: true
  autoscaleMin: 1
  autoscaleMax: 5
  replicaCount: 1
  rollingMaxSurge: 100%
  rollingMaxUnavailable: 25% 

  hub: "" 
  tag: "" 

  # Can be a full hub/image:tag
  image: pilot
  traceSampling: 1.0 

  # Resources for a small pilot install
  resources:
    requests:
      cpu: 500m
      memory: 2048Mi 

  env: {} 

  cpu:
    targetAverageUtilization: 80
```
Similarly, we can get the values of other Helm charts. To apply the configuration updates to individual chart installations, we would create a separate YAML file with the configuration value we want to update. Then, use the install command with the -f flag to install the individual chart with the provided configuration settings:

helm install istiod istio/istiod -n istio-system -f my-config-values.yaml


