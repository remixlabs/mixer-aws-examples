# EKS example (full)

This example contains a Helm chart for deploying the service, as well as an
example EKS CloudFormation template, in `cluster.yaml`, for a cluster that we've
used to test it.

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

