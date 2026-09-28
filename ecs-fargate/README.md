# ecs-fargate

This directory shows an example of deploying Remix Server in an ECS Fargate
cluster, using an EFS volume for the database disk.

Whereas deploying to EKS can use generic Kubernetes tooling like Helm, ECS is
its own beast, so the example here is an AWS-specific CloudFormation template.

## EFS vs. EBS for the persistent volume

Fargate can't mount an EBS volume as simply as the EKS chart does (a
pre-created EBS volume, statically bound via `PersistentVolume`/
`PersistentVolumeClaim`, with pods pinned to its AZ via `nodeSelector`).
Fargate's own EBS-volume-attachment feature is newer and geared toward a new
volume per task launch, not "reattach this exact volume every time the
service restarts." **EFS** is the mature, first-class Fargate option for a
volume that needs to survive task replacement: mount it via
`efsVolumeConfiguration`, and every new task just re-mounts the same
filesystem — no AZ pinning needed, since EFS is regional.

The one thing to watch: EFS is a network filesystem (NFS), not a block
device. Performance may suffer for database-heavy tasks.

## Prerequisites

- An AWS account/region with a VPC and at least two subnets in different AZs
  (the default VPC works fine for a test).
- Cross-account access to pull the private `mixer` image from
  `250233190882.dkr.ecr.us-east-1.amazonaws.com` — arrange this with Remix
  before deploying, or the task will fail with `CannotPullContainerError`.
- If the task will run without a public IP (`AssignPublicIp: DISABLED`), the
  subnets need a NAT gateway or VPC interface/gateway endpoints as noted
  below — otherwise it can't pull the image, ship logs, or mount EFS.
- An IAM identity to run `aws cloudformation`/`ecs`/etc. against this stack.
  `admin-iam-policy.json` in this directory documents the minimal set of
  permissions the template needs (CloudFormation, IAM role management, EC2
  security groups and ENIs, ECS, EFS, Logs). It also grants
  `iam:CreateServiceLinkedRole` scoped to `AWSServiceRoleForECS` — on a
  genuinely fresh account, ECS's first `CreateService` call creates that
  service-linked role using the caller's own permissions, so without this the
  stack fails on the `Service` resource the first time ECS is used in the
  account. It's a one-time cost per account; harmless to leave granted after.
  The EFS statement also carries several `Describe*` actions
  (`DescribeFileSystemPolicy`, `DescribeLifecycleConfiguration`,
  `DescribeBackupPolicy`, `DescribeReplicationConfigurations`) that the
  template never exercises directly — CloudFormation's `AWS::EFS::FileSystem`
  resource handler reads back that config as part of its own create/update/
  delete flow regardless, and 403s on whichever one it happens to call if
  it's missing. For a disposable test account it's not
  worth hand-scoping this — just attach `AdministratorAccess` — but the
  policy is here so the example documents what a real deployment actually
  requires (note `PowerUserAccess` alone is *not* enough, since it excludes
  IAM and this template creates roles via `CAPABILITY_NAMED_IAM`).

## Deploying

```sh
aws cloudformation create-stack \
  --stack-name mixer-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters file://parameters.example.json
```

(Use `update-stack` with the same arguments for subsequent changes.)

Required inputs: `VpcId`, `SubnetOne`, `SubnetTwo`.

This setup assumes the chosen subnets can reach ECR, CloudWatch Logs, and EFS
(either via a NAT gateway, or VPC interface/gateway endpoints for
`ecr.api`/`ecr.dkr`/`logs`/`elasticfilesystem`, plus the `s3` gateway endpoint
for image layers).

## Persistent signing key

By default the server generates a new signing key every time it starts, so
anything it signed before a restart or redeploy stops validating. To keep
the key stable, create it once in Secrets Manager and pass the secret's ARN
as `ServerKeySecretArn`. ECS then injects it into the container as
`MIXER_SERVER_KEY` at launch. The template and the task definition only
contain the ARN, never the key.

The key is a P-256 private key in PEM format, base64-encoded. Generate it
and pipe it straight into Secrets Manager. That way it's never written to
disk, never lands in shell history, and never shows up in a process
listing:

```sh
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 \
  | base64 | tr -d '\n' \
  | aws secretsmanager create-secret \
      --name mixer-dev/server-key \
      --query ARN --output text \
      --secret-string file:///dev/stdin
```

Add the printed ARN to your parameters file:

```json
  { "ParameterKey": "ServerKeySecretArn", "ParameterValue": "arn:aws:secretsmanager:us-east-1:123456789012:secret:mixer-dev/server-key-AbCdEf" },
```

Notes:

- The secret lives outside the stack on purpose. Deleting or recreating the
  stack doesn't touch it, so a rebuilt deployment keeps the same key.
- When `ServerKeySecretArn` is set, the stack grants the task *execution*
  role `secretsmanager:GetSecretValue` on that one secret, and nothing
  else. If you encrypt the secret with a customer-managed KMS key instead
  of the default `aws/secretsmanager` key, also give that role
  `kms:Decrypt` on the KMS key.
- The identity that creates the secret needs `secretsmanager:CreateSecret`.
  That permission isn't in `admin-iam-policy.json`, because the stack itself
  never calls it.
- ECS reads the secret only when a task starts. If you rotate the key, run
  `aws ecs update-service --force-new-deployment` to pick up the new value.
- Anyone who can read the secret, or exec into the running container, can
  get the key. Scope IAM access to both accordingly.

## Deleting the stack

The EFS filesystem holding the agent databases (`AgentDbFileSystem`) has
`DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`. Deleting the
stack, or a stack update that would replace the filesystem, leaves it
behind with its data intact rather than destroying it. A new stack does
*not* pick it up again: it creates a fresh, empty filesystem. To get rid of
a retained filesystem (e.g. after a throwaway test), delete it yourself.
It's tagged `Name=agent-db-${Env}`:

```sh
aws efs describe-file-systems \
  --query "FileSystems[?Name=='agent-db-dev'].FileSystemId"
aws efs delete-file-system --file-system-id fs-0123456789abcdef0
```

The filesystem also has EFS automatic backups turned on (`BackupPolicy`),
so AWS Backup takes daily snapshots into the account's default EFS backup
vault. These recovery points are kept independently of the filesystem, and
by default for 35 days, so they remain after the filesystem is deleted.
They're billed as AWS Backup storage.

## Outputs

`ClusterName`, `ServiceName`, `TaskDefinitionArn`, `EfsFileSystemId`,
`EfsAccessPointId`, `LogGroupName`.

## Reaching the service from a browser

The template has no load balancer -- you decide how to expose it. Two ways to
test:

- **Public IP, locked to one CIDR (fast, what's wired up by default):** set
  `AssignPublicIp` to `ENABLED` and `IngressCidr` to your own IP in `/32`
  form (`curl -s https://checkip.amazonaws.com`), and put the service in
  public subnets. After the service stabilizes, look up the task's public IP
  (`describe-tasks` → ENI → `describe-network-interfaces`) and browse
  `http://<that-ip>:8000/`. The IP changes on every task replacement.
- **Fully private, tunnel in:**
  See `private-access-via-ssm.md` for a sketch of the SSM port-forwarding
  approach.
