# ecs-fargate

CloudFormation analog of `../eks-simple`: runs the `mixer` server on ECS
Fargate with a persistent volume mounted at `/dbs` for its custom database.

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
