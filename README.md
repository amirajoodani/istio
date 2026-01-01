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
