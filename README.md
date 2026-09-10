# mixer-aws-examples

This repository explains how to self-host Remix Server and provides example
configurations for deploying to AWS EKS and ECS.

The overall Remix platform documentation is on our [Notion
page](https://curious-turnover-84b.notion.site). There is a page for this
[service
specifically](https://curious-turnover-84b.notion.site/Cloud-Agent-Server-1061d464528f80f18e46dd27895aeda2),
but for API usage, the best thing is probably the [deployed Swagger
docs](https://agt.remixlabs.com/v1/swagger/#/).

## Remix Server overview

Remix Server's deployment footprint is simple: it's a single-node server defined
by a Docker image, which requires an attached persistent disk to store its
built-in custom database. Once these are created and provided appropriate
resources, the service itself can be managed from Remix Desktop.

The server provides its own authentication mechanisms, and can also serve as the
authentication server for Remix Desktop. Any identity provider that defines an
OAuth integration can be used, and access control to Remix resources is managed
within the server itself. Service authentication is bootstrapped by using the
desktop app to generate a key pair, and providing the public key to the server
environment. The desktop app can can authenticate and set up the identity
provider for OAuth sign-in.

Remix Server can be deployed into any cloud infrastructure that can run Docker
containers and provide a persistent disk: EKS or ECS in AWS, GCP's GKE, etc. We
also have a configuration that runs in Snowflake's
[Snowpark](https://www.snowflake.com/en/product/features/snowpark/) Container
Services environment.

## Example configurations

Here are five example deployment configurations; which is most useful will
depend on your existing infrastructure. ECS is the simplest to get up and
running and requires the fewest prerequisites and configuration.

The ECS configurations are provided as CloudFormation templates, while the EKS
examples are Helm charts (which could be straightforwardly ported to other
Kubernetes environments).

[ecs-fargate](ecs-fargate/README.md) provides a CloudFormation template for a
complete ECS deployment, including the cluster definition, management roles,
etc. It documents two ways to reach the (by default, fully private) service:
a locked-down public IP for quick testing, and an SSM port-forwarding tunnel
for fully private access without any public exposure.

[ecs-fargate-client-vpn](ecs-fargate-client-vpn/README.md) is the same ECS
deployment, configured for the more typical customer shape: internal users
reaching the server through an existing AWS Client VPN into the VPC, rather
than a public IP or an ad hoc tunnel. It also documents how to stand up a
Client VPN endpoint from scratch in a test account.

[ecs-fargate-vpn-plus-alb](ecs-fargate-vpn-plus-alb/README.md) is the same
ECS deployment reachable two ways at once: an AWS Client VPN for normal use,
plus a public Application Load Balancer that exposes a small, explicit set
of URL paths (webhooks, an OAuth callback, etc.) to callers that can't be on
the VPN. Unlike `ecs-fargate-client-vpn`, it creates its own VPC, plus a
public Route 53 hosted zone and DNS-validated ACM certificate, so it's
self-contained aside from delegating a subdomain's NS records and configuring
the VPN itself.

[eks-full](eks-full/README.md) provides a complete EKS example configuration,
including a load balancer, cluster definition, load balancer, ingress rules,
etc.

[eks-simple](eks-eimple/README.md) is a stripped-down version that assumes you
already have a cluster and want to work out the networking yourself. It just
includes the service and deployment definitions, plus a persistent volume claim
for the database disk.

### Snowflake Snowpark

Contact us for more information about running Remix Server in Snowpark. For that
environment, we can provide privately listed Snowflake Marketplace app that
installs Remix Server directly into your Snowflake account, so that the service
runs entirely within your Snowflake trust boundary, alongside your data. Remix
Desktop can connect to the workspace server via a Snowflake OAuth integration.

If you want to use Remix with Snowflake without running the server in Snowpark,
this is also possible via a typical OAuth connection between Remix Server and
Snowflake. Running Remix server in your own cloud environment will likely be
lower-cost and give you more control, but require greater configuration and
admin overhead.

## The Remix Server Docker image

The `mixer` Docker image is available in a private AWS ECR repository, at
`250233190882.dkr.ecr.us-east-1.amazonaws.com/mixer`. There are amd64 and arm64
versions available. Each has a series of releases tagged with a build number
(current latest: `12350` and `12350-arm64`). You can configure the release to
either pin to a particular build, or use the mutable `latest`/`latest-arm64`
tags.

You will need to be granted access to that image; we can do that with the ARN of
an AWS IAM role or account that will pull it, or contact us for alternative
approaches.

