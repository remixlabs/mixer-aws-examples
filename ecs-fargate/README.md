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

## Deploying

```sh
aws cloudformation create-stack \
  --stack-name mixer-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters file://parameters.example.json
```

(use `update-stack` with the same arguments for subsequent changes.)

Required inputs you must supply (no sensible default exists): `VpcId`,
`SubnetOne`, `SubnetTwo`. Everything else defaults to the same values as
`eks-simple/values.yaml`.

Assumes the chosen subnets can reach ECR, CloudWatch Logs, and EFS (either
via a NAT gateway, or VPC interface/gateway endpoints for
`ecr.api`/`ecr.dkr`/`logs`/`elasticfilesystem`, plus the `s3` gateway
endpoint for image layers).

## Outputs

`ClusterName`, `ServiceName`, `TaskDefinitionArn`, `EfsFileSystemId`,
`EfsAccessPointId`, `LogGroupName`.
