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

## Persistent signing key

By default the server generates a new signing key every time it starts. To
keep the key stable, store it in a Kubernetes Secret and set
`server_key_secret` to that Secret's name. The chart injects its
`MIXER_SERVER_KEY` entry into the pod. The key is a P-256 private key in
PEM format, base64-encoded. Generate it straight into the Secret, so it
never touches disk or `values.yaml`:

```sh
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 \
  | base64 | tr -d '\n' \
  | kubectl create secret generic mixer-server-key \
      --from-file=MIXER_SERVER_KEY=/dev/stdin
```

Then deploy with `--set server_key_secret=mixer-server-key`. Kubernetes
Secrets are only base64-encoded at rest by default. On EKS, turn on
[envelope encryption](https://docs.aws.amazon.com/eks/latest/userguide/enable-kms.html)
with KMS and restrict RBAC `get` on Secrets. Alternatively, keep the key in
AWS Secrets Manager and sync it in with the Secrets Store CSI Driver or
External Secrets Operator.
