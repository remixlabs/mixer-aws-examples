# EKS (simple)

This example is just a subset of the resources from [EKS
(full)](../eks-full/README.md), stripped down to the core of what is specific to
Remix: the service definition itself, and the persistent volume claim for the
database disk. The cluster, load balancer, and ingress mechanics are left as
exercises for the reader.

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
