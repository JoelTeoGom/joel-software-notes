# Security and networking

How attacks use the network underneath an application, and the defenses that hold up: what a request really does on the wire, who resolves which name, and where to put the check so it can't be bypassed. Mostly Go and Kubernetes, with real systems (webhooks, schedulers) as running examples.

## Learning path

| # | Topic | You'll understand |
|---|---|---|
| 01 | [SSRF](01-ssrf/README.md) | what SSRF is, read vs blind, why URL validation fails (DNS rebinding), checking the IP in `net.Dialer.Control`, forward proxies and `CONNECT`, egress proxies like Stripe's Smokescreen |

## Glossary

| Term | Meaning |
|---|---|
| SSRF | Server-Side Request Forgery: making a server send requests to destinations the attacker chooses |
| Blind SSRF | SSRF where the attacker never sees the response, but still causes side effects or learns from errors and timings |
| DNS rebinding | Changing a DNS answer between the moment it's checked and the moment it's used |
| TOCTOU | Time of check vs time of use: a bug where what you checked is not what you use |
| Metadata service | `169.254.169.254` in AWS, GCP and Azure: answers instance details and credentials to requests from the machine |
| IMDSv2 | AWS metadata service that requires a session token obtained with a `PUT` |
| Forward proxy | An intermediate server that makes outbound connections on behalf of clients |
| `CONNECT` | HTTP method that asks a proxy to open a TCP tunnel to `host:port` |
| Egress proxy | A forward proxy that all outbound traffic of a platform must go through, to check and audit it |
| Link-local | `169.254.0.0/16` and `fe80::/10`: addresses only valid on the local network segment |
| CGNAT | `100.64.0.0/10`: shared address space, used by some clouds and VPNs as internal space |
