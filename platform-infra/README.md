# Platform and infrastructure

How code becomes a running container, and how a request finds it: registries, GitOps and the Kubernetes control plane, AWS regions, the full path of a request through DNS, load balancers and ingresses, egress, sidecars and CDNs. Mostly AWS and Kubernetes, with my homelab and Atlassian Cloud as running examples.

## Learning path

Read 01-02 for "how code gets deployed" and 03-08 for "how a request gets served". Each block builds on the previous topic.

| # | Topic | You'll understand |
|---|---|---|
| 01 | [Container registry](01-container-registry/README.md) | what an image physically is (manifest + layers), how push/pull work over HTTPS, tags vs digests |
| 02 | [GitOps: from a commit to a running pod](02-gitops-from-commit-to-pod/README.md) | push vs pull, the 14 steps from `git push` to a serving pod, the manifest repo, Argo CD, expand/contract migrations, ECS as the push-based alternative |
| 03 | [AWS regions and AZs](03-aws-regions-and-azs/README.md) | what a region and an AZ are, global vs regional vs zonal resources, the price of multi-region |
| 04 | [The path of a request in AWS](04-path-of-a-request-in-aws/README.md) | DNS with Route 53, the Internet Gateway, the ALB, the ingress, a tenant lookup and a hop to another region, with Atlassian as the example |
| 05 | [Load balancers](05-load-balancers/README.md) | L4 vs L7 vs API gateway, CLB/ALB/NLB, why load balancers have to scale, algorithms and the p99, why L4 breaks gRPC |
| 06 | [Ingress and egress](06-ingress-and-egress/README.md) | the Ingress object vs the controller, Gateway API, programming an ingress, when and how to control egress |
| 07 | [Sidecars](07-sidecars/README.md) | why "the client" of a service is hard, what a sidecar is, library vs sidecar |
| 08 | [CDN](08-cdn/README.md) | edge caches, hit vs miss, `Cache-Control`, the cache key, versioned assets vs invalidation |

## The one idea that connects everything

Almost every topic is about **a layer that knows something the layer before it doesn't**:

- DNS knows which region is closest to the user, but not where the user's data is.
- The load balancer knows which instances are healthy, but (at L4) nothing about HTTP.
- The ingress knows the tenant and asks a directory where it lives.
- The kubelet doesn't know about Deployments; it only runs the pods bound to its node.
- Argo CD doesn't know about commits; it only compares the whole desired state with the live one.

Understanding a system is mostly knowing what each layer can see.

## Glossary

| Term | Meaning |
|---|---|
| OCI | Open Container Initiative: the standard image format and registry API |
| Manifest / blob | An image's JSON description / any content stored by its `sha256` (layers, config) |
| GitOps | The cluster pulls its desired state from Git and continuously reconciles it |
| CRI / CNI | Container Runtime Interface (kubelet ↔ containerd) / Container Network Interface (pod networking) |
| Region / AZ | A geographic area / an isolated group of data centers inside it |
| VPC | Virtual Private Cloud: your isolated network in AWS |
| IGW / NAT gateway | The VPC's door to the internet / outbound-only internet access for private subnets |
| ALB / NLB / CLB | AWS Application (L7) / Network (L4) / Classic load balancer |
| Ingress / egress | Traffic into / out of your system |
| Sidecar | A helper container next to the app, sharing its network |
| CDN / PoP | Content delivery network / point of presence (an edge location) |
| TTL | Time to live: how long a cached answer (DNS, HTTP) is valid |
