# mixer-aws-examples

Examples configurations to self-host Remix Server in AWS.

The overall Remix platform documentation is on our [Notion
page](https://curious-turnover-84b.notion.site). There is a page for this
[service
specifically](https://curious-turnover-84b.notion.site/Cloud-Agent-Server-1061d464528f80f18e46dd27895aeda2),
but for usage the best thing is probably the [deployed Swagger
docs](https://agt.remixlabs.com/v1/swagger/#/).

This repo contains a Helm chart for deploying the service, as well as an example
EKS CloudFormation template in `cluster.yaml` that we've used to test it.

The Helm chart includes:

- `templates/service.yaml`: the basic Kubernetes service, including a port
  mapping
- `templates/deployment.yaml`: the actual spec for the single managed pod,
  including the environment variables for configuring it
- `templates/pvc.yaml`: a PersistentVolumeClaim spec for a managed EBS volume,
  which the service's pod mounts as a database volume. Other means of attaching
  a persistent volume would be fine, as long as they can be configured to live
  longer than the pods themselves.
- `templates/ingress.yaml`: an ALB ingress configuration, intended to be used
  alongside an ACM managed certificate for TLS. A managed cert is defined in
  the example cluster template, but you still need to wire up DNS for it to
  work. (That is, create the Helm release, get the endpoint for the ALB, then
  CNAME whatever public (sub)domain you want to that.) This is for a publicly
  exposed server; for internal-only access, other configurations are possible.
- `values.yaml`: defines default values for a variety of configuration
  parameters affecting the service at runtime

The repo also includes:

- `cluster.yaml`: a CloudFormation template defining an EKS cluster and
  associated resources, as an example of an environment where you might run the
  service

## The actual Docker image

The `mixer` Docker image is available in a private AWS ECR repository, at
`250233190882.dkr.ecr.us-east-1.amazonaws.com/mixer`. There are amd64 and arm64
versions available. Each has a series of releases tagged with a build number
(current latest: `12350` and `12350-arm64`). You can configure the release to
either pin to a particular build, or use the mutable `latest`/`latest-arm64`
tags.

You will need to be granted access to that image; we can do that with the ARN of
an AWS IAM role that will pull it, or contact us for alternative approaches.

## Other deployment targets

- `eks-simple/`: a minimal Helm chart (Deployment + Service + PVC only, no
  ingress) for running the service on an existing EKS cluster.
- `ecs-fargate/`: a CloudFormation template for running the service on ECS
  Fargate instead of Kubernetes, for customers who don't run EKS. Storage
  uses EFS rather than EBS; see `ecs-fargate/README.md` for the full mapping
  from the Helm chart's resources.
