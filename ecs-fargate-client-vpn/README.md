# ecs-fargate-client-vpn

This is the same deployment as the [`ecs-fargate`](../ecs-fargate/README.md) —
"mixer" on ECS Fargate with an EFS-backed `/dbs` volume — configured for the
case where users reach it through an **AWS Client VPN** already terminating in
their VPC, instead of a public IP or an SSM tunnel.

## Changes from `ecs-fargate`

- **`IngressCidr` → `ClientVpnTargetSubnetCidrOne`/`Two`.** AWS Client VPN
  NATs client traffic before it reaches the VPC: a connected user's packets
  arrive at the task with a source IP from whichever subnet the Client VPN
  endpoint is associated with (a *target-network* subnet — not necessarily
  `SubnetOne`/`SubnetTwo`, since the endpoint can be associated with
  different subnets than the ones the mixer task runs in), not from the
  VPN's own client CIDR block. `ServiceSecurityGroup`'s ingress rules are
  keyed off those target-network subnets' CIDRs accordingly.
- **`AssignPublicIp` still exists but is DISABLED.** `AssignPublicIp=ENABLED` is
  left in only as a fallback debugging escape hatch.
- **`admin-iam-policy.json` adds one read-only statement** (`ClientVpnLookup`)
  covering `ec2:DescribeClientVpn*` — enough to look up the existing
  endpoint's target-network associations before deploying. This template
  does **not** create, associate, or authorize the Client VPN endpoint
  itself — see the assumption below.
- **No VPC routing changes were needed beyond the security group.** Because
  Client VPN target-network association and the endpoint's own internal
  route table are per-subnet, and this template assumes the endpoint is
  already associated with subnets in `VpcId` (intra-VPC routing to
  `SubnetOne`/`SubnetTwo` is automatic via each route table's implicit
  `local` route, regardless of which subnets those are), no VPC route table
  changes or `AWS::EC2::ClientVpn*` resources are required here.
- **Adds ECS Service Discovery (AWS Cloud Map)** so the task is reachable at
  a stable name — `mixer.mixer-${Env}.internal` — instead of an ENI IP that
  changes on every task replacement (see "Reaching the service" below).
  Creating a Cloud Map *private DNS* namespace creates a Route 53 private
  hosted zone under the hood, so the deploying principal needs Route 53
  permissions in addition to `servicediscovery:*` — see the
  `Route53ForServiceDiscoveryPrivateZone` statement in
  `admin-iam-policy.json` (this repo file is a reference; make sure the IAM
  policy actually attached to your deploying identity includes it).

## Assumption: the Client VPN already exists

This template assumes an AWS Client VPN endpoint is already:

1. Associated with a subnet in `VpcId` (doesn't have to be `SubnetOne` or
   `SubnetTwo` specifically — any subnet in the same VPC routes to them).
2. Authorized (via a Client VPN authorization rule) to reach `VpcId`'s CIDR,
   or at least the specific subnets the mixer task runs in.

If that's not true yet, see [`client-vpn-setup.md`](client-vpn-setup.md) for
how to stand one up from scratch in a test account.

## Deploying

Same as the base example:

```sh
aws cloudformation create-stack \
  --stack-name mixer-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters file://parameters.example.json
```

Required inputs: `VpcId`, `SubnetOne`, `SubnetTwo`, and
`ClientVpnTargetSubnetCidrOne`/`Two` — the CIDRs of the subnets the Client
VPN endpoint is associated with (not the VPN's client CIDR block). Find the
subnet IDs with `aws ec2 describe-client-vpn-target-networks
--client-vpn-endpoint-id <id>`, then their CIDRs with `aws ec2
describe-subnets --subnet-ids <id> <id>`.

Same egress prerequisites as the base example apply unchanged: the task's
subnets need a NAT gateway or VPC interface/gateway endpoints for
ECR/CloudWatch Logs/EFS.

If `SubnetOne`/`SubnetTwo` don't already have one, put the NAT gateway in a
**different, still-public** subnet — not in `SubnetOne` or `SubnetTwo`
themselves. A NAT gateway only works if its own subnet still routes to an
Internet Gateway; if you route `SubnetOne`/`SubnetTwo` to a NAT gateway that
lives in one of them, you've cut off that subnet's own path to the internet,
and the NAT gateway (and everything behind it, including ECR pulls) times
out. Use a separate route table for `SubnetOne`/`SubnetTwo` pointing at the
NAT gateway, and leave the NAT gateway's own subnet on the original
IGW-routed table.

## Persistent signing key

Same as the base example: generate the key into Secrets Manager and pass the
secret's ARN as the optional `ServerKeySecretArn` parameter. See
[Persistent signing key](../ecs-fargate/README.md#persistent-signing-key)
in the `ecs-fargate` README for the command and the details.

## Reaching the service

1. Connect to the Client VPN (AWS-provided VPN client, Tunnelblick, OpenVPN
   Connect, etc. — whatever the customer's existing profile uses).
2. Browse `http://mixer.mixer-${Env}.internal:8000/` (e.g.
   `http://mixer.mixer-dev.internal:8000/` for `Env=dev`) — no need to look
   up the task's IP.

This works because the template creates a Cloud Map private DNS namespace
(`mixer-${Env}.internal`, associated with `VpcId`) and registers the ECS
service with it (`ServiceRegistries` on `AWS::ECS::Service`). ECS keeps the
namespace's `mixer` A record pointed at whatever the current task's IP is,
updating it automatically on every task replacement — the caller never
needs the raw IP. The record uses a 10s TTL so failover after a task
replacement is fast.

This requires the Client VPN endpoint to have `DnsServers` set to the VPC's
own resolver — **not the default.** If name resolution doesn't work (or you
hit anything else connecting through the VPN), see
[`client-vpn-setup.md`](client-vpn-setup.md)'s "Troubleshooting" section for
how to check/fix that and other common pitfalls.

## Deleting the stack

As in the base example, the EFS filesystem is retained when the stack is
deleted, and has daily automatic backups turned on. See
[Deleting the stack](../ecs-fargate/README.md#deleting-the-stack) in the
`ecs-fargate` README.

## Outputs

`ClusterName`, `ServiceName`, `TaskDefinitionArn`, `EfsFileSystemId`,
`EfsAccessPointId`, `LogGroupName`, `ServiceDnsName`.
