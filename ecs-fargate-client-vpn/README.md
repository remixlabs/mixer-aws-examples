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

## Reaching the service

1. Connect to the Client VPN (AWS-provided VPN client, Tunnelblick, OpenVPN
   Connect, etc. — whatever the customer's existing profile uses).
2. Look up the task's current private IP (it changes on every task
   replacement, same caveat as the base example):
   ```sh
   TASK_ARN=$(aws ecs list-tasks --cluster agents-cluster --service-name agent-server-dev --query 'taskArns[0]' --output text)
   ENI_ID=$(aws ecs describe-tasks --cluster agents-cluster --tasks "$TASK_ARN" \
     --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text)
   TASK_IP=$(aws ec2 describe-network-interfaces --network-interface-ids "$ENI_ID" \
     --query 'NetworkInterfaces[0].PrivateIpAddress' --output text)
   ```
3. Browse `http://$TASK_IP:8000/` directly — no tunnel or bastion needed,
   since the VPN itself puts your machine on a routable path to the VPC.

If the connection hangs rather than being actively refused, double check
`ClientVpnTargetSubnetCidrOne`/`Two` — they need to be the associated
subnets' CIDRs, not the VPN's client CIDR block.

## Outputs

`ClusterName`, `ServiceName`, `TaskDefinitionArn`, `EfsFileSystemId`,
`EfsAccessPointId`, `LogGroupName`.
