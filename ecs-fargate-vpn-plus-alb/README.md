# ecs-fargate-vpn-plus-alb

Runs the "mixer" server on ECS Fargate, reachable two ways at once: a
private **AWS Client VPN** for normal use, plus a public **Application
Load Balancer** that exposes a small, explicit set of URL paths (webhooks,
an OAuth callback, etc.) to callers that can't be on the VPN. Unlike
[`ecs-fargate-client-vpn`](../ecs-fargate-client-vpn/README.md), which
assumes a VPC (and a Client VPN endpoint already associated with it)
already exists, this stack **creates its own VPC from scratch** — a public
and private subnet pair in each of two AZs, an Internet Gateway, a NAT
Gateway, and the route tables wiring it together. To make the ALB's TLS
story self-contained too, the stack also creates a public Route 53 hosted
zone and a DNS-validated ACM certificate for a subdomain you own — you only
need to delegate that subdomain's NS records at your existing DNS provider,
and point a Client VPN endpoint at the VPC this stack creates, nothing else
external.

The stack includes its own VPC to make the example self-contained. In
particular, the default VPC won't work, because the ALB target group requires
that the target's AZ is one the the ALB has a subnet in. So if you want private
subnets for the service but public for the ALB, you need at least two subnets in
at least AZ's, which is what this VPC sets up. (The default VPC has one subnet
for each AZ.)

## VPN vs. ALB

The VPN path and the ALB path serve different callers of the same service:

- **Client VPN + internal DNS** (`mixer.mixer-${Env}.internal`) — for
  Remix Desktop / interactive users. Full access to everything the app
  serves.
- **Public ALB** (`https://${SubdomainName}/...`) — for callers that are
  never going to be VPN-connected, like a webhook sender (e.g. Google
  Calendar push notifications) or an OAuth provider's redirect. Only the
  specific paths wired up as `AWS::ElasticLoadBalancingV2::ListenerRule`
  resources are reachable; everything else gets a flat 403 from the ALB's
  default action, without ever reaching a task. See `template.yaml`'s
  `HttpsListener` and the `WebhooksListenerRule`/`OauthCallbackListenerRule`
  examples — add one rule per real path you need, and don't change the
  listener's default action to a forward.

Both paths route to the same ECS service and the same task; `mixer`
differentiates by URL path internally, same as any app with both public and
private routes.

## What's in the VPC

- **`Vpc`** (default `10.0.0.0/16`) with DNS support and DNS hostnames both
  enabled — required for the Cloud Map private DNS namespace (and for
  Client VPN's pushed DNS resolver to work at all).
- **`PublicSubnetOne`/`PublicSubnetTwo`** (one per AZ) — routed to
  **`InternetGateway`** via `PublicRouteTable`. Host the ALB, and the single
  **`NatGateway`** (in `PublicSubnetOne` only — cheaper for an example; a
  real deployment wanting per-AZ egress resilience would give each AZ its
  own NAT Gateway instead of sharing one across both).
- **`SubnetOne`/`SubnetTwo`** (one per AZ, matching the public pair's AZs) —
  routed to the NAT Gateway via `PrivateRouteTable`. Host the Fargate task
  and the EFS mount targets, same as the base example.
- `ServiceSecurityGroup` trusts `SubnetOne`/`SubnetTwo`'s own CIDR blocks
  directly (`!GetAtt SubnetOne.CidrBlock`) instead of taking them as a
  separate parameter you'd have to discover and feed back in — since this
  stack defines those subnets itself, it already knows their CIDRs. Point
  the Client VPN endpoint's target-network association at these same two
  subnets (see `client-vpn-setup.md`) and there's nothing left to
  reconcile.

## Changes from `ecs-fargate-client-vpn`

- **The VPC itself** — see above.
- **New `SubdomainName` parameter** and two new resources —
  `AWS::Route53::HostedZone` and `AWS::CertificateManager::Certificate`
  (`ValidationMethod: DNS`, validated against that hosted zone
  automatically) — so the ALB has a real public hostname and a publicly
  trusted cert without any manual ACM console steps.
- **`AlbSecurityGroup`** (443/80 open to the internet) and one added ingress
  rule on `ServiceSecurityGroup` trusting `AlbSecurityGroup` on
  `ContainerPort` — the ALB reaches the task the same way the VPN's
  private subnets do, as a distinct trusted source.
- **`LoadBalancer`, `TargetGroup`, `HttpListener` (redirects to 443),
  `HttpsListener` (default action: fixed 403), and two example
  `ListenerRule`s** wired into `Service.LoadBalancers`.
- **The admin IAM policy is split into three files** — a single combined
  policy exceeds IAM's 6,144-character limit for a customer-managed policy,
  so attach all three to the deploying principal; none alone is sufficient
  for this template:
  - [`admin-iam-policy-core.json`](admin-iam-policy-core.json) — everything
    the base example already needed: CloudFormation, task/execution IAM
    roles, EC2 describe actions + security groups, ECS, the private Cloud
    Map zone, EFS, logs.
  - [`admin-iam-policy-networking.json`](admin-iam-policy-networking.json) —
    creating the VPC itself: subnets, route tables, the Internet Gateway,
    the NAT Gateway and its Elastic IP.
  - [`admin-iam-policy-public-endpoint.json`](admin-iam-policy-public-endpoint.json) —
    the public hosted zone, ACM cert lifecycle, the ELB service-linked
    role, and the `elasticloadbalancing:*` actions for the load
    balancer/target group/listeners/rules.

  Two spots in `admin-iam-policy-public-endpoint.json` grant actions this
  template never calls directly, because the underlying AWS resource
  handler calls them itself as part of its own create/update/delete flow —
  same story as the EFS `Describe*` actions already called out in the base
  example's README:
  - `route53:ListQueryLoggingConfigs` — `AWS::Route53::HostedZone`'s delete
    handler checks for a query-logging config to clean up first, and 403s
    without it even though this stack never configures one.
  - `Ec2LookupsForAlbProvisioning` (`DescribeInternetGateways`,
    `DescribeAccountAttributes`, `DescribeAddresses`, `DescribeTags`) —
    what `CreateLoadBalancer` needs to validate the subnets you gave it
    actually route to an Internet Gateway.

Everything else — EFS volume, Cloud Map private namespace, task
definition — is identical to `ecs-fargate-client-vpn`; see that README for
the parts this one doesn't repeat.

## The Client VPN endpoint comes after this stack, not before

This is the one place the dependency direction flips relative to
`ecs-fargate-client-vpn`. That example assumes the VPC and its Client VPN
endpoint already exist, and takes their IDs/CIDRs as parameters. Here,
*this* stack creates the VPC — so the Client VPN endpoint has to be created
(or re-pointed) afterward, targeting this stack's own `VpcId`/`SubnetOneId`/
`SubnetTwoId` Outputs. [`client-vpn-setup.md`](client-vpn-setup.md) walks
through this in a test account, and starts from those Outputs rather than
from a pre-existing VPC.

## Deploying

The stack creation has one real chicken-and-egg: "the hosted zone doesn't
exist until the stack creates it" vs. "the certificate won't validate until
that zone's NS records are delegated." The VPC/subnet/routing resources
don't have this problem — they're ordinary, fast-creating resources with no
external dependency, so they're done long before the certificate is the
thing you're waiting on.

```sh
aws cloudformation create-stack \
  --stack-name mixer-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters file://parameters.example.json
```

This call returns immediately, but the stack itself will sit at
`CREATE_IN_PROGRESS` for a while — `PublicHostedZone` is created almost
instantly, but `Certificate` blocks until ACM can see a DNS validation
record it wrote into that zone, which can't happen until the zone is
actually delegated. While it's sitting there:

1. Get the hosted zone's ID and name servers — the zone resource already
   exists even though the stack as a whole hasn't finished:
   ```sh
   ZONE_ID=$(aws cloudformation describe-stack-resource \
     --stack-name mixer-dev --logical-resource-id PublicHostedZone \
     --query 'StackResourceDetail.PhysicalResourceId' --output text)
   aws route53 get-hosted-zone --id "$ZONE_ID" --query 'DelegationSet.NameServers'
   ```
2. At your parent domain's DNS provider, create NS records for
   `SubdomainName` pointing at those four name servers.
3. Wait for that delegation to propagate (usually minutes; can be longer
   depending on your provider's/registrar's own TTLs) and for ACM's
   validator to poll and see it. CloudFormation keeps waiting on the
   `Certificate` resource during ordinary propagation delay — you don't need
   to re-run anything.
4. Once the certificate validates, the rest of the stack (ALB, listeners,
   rules, the alias record) creates normally and the stack reaches
   `CREATE_COMPLETE`.

If you'd rather not straddle a long-running `create-stack` call, you can
split this into two runs instead: first deploy with only `PublicHostedZone`
reachable (e.g. comment out everything from `Certificate` on down), fetch
its name servers, delegate them, confirm delegation with `dig NS
$SUBDOMAIN`, then uncomment the rest and `update-stack`. The single-call
version above is simpler when you don't mind the wait.

Once `CREATE_COMPLETE`, follow [`client-vpn-setup.md`](client-vpn-setup.md)
to stand up the Client VPN endpoint against this stack's own VPC.

## Reaching the service

- **Via VPN**: connect to the Client VPN (see `client-vpn-setup.md`),
  browse `http://mixer.mixer-${Env}.internal:8000/`.
- **Via the public ALB**, only for the specific paths wired up:
  ```sh
  curl https://<SubdomainName>/webhooks/whatever
  ```
  Anything else (`curl https://<SubdomainName>/`, for instance) gets a 403.

## Adding a new exposed path

Add another `AWS::ElasticLoadBalancingV2::ListenerRule` pointed at
`HttpsListener`, with its own `Priority` (rules are evaluated in ascending
priority order, first match wins) and a `path-pattern` condition, forwarding
to the existing `TargetGroup`. Don't add a broad pattern like `/*` — that
defeats the whole reason for using path-based rules instead of just exposing
the ALB's default action.

## Outputs

`ClusterName`, `ServiceName`, `TaskDefinitionArn`, `EfsFileSystemId`,
`EfsAccessPointId`, `LogGroupName`, `ServiceDnsName` (same as the base
example), plus `VpcId`, `SubnetOneId`, `SubnetTwoId`, `PublicSubnetOneId`,
`PublicSubnetTwoId` (needed for `client-vpn-setup.md`), `PublicHostedZoneId`,
`PublicHostedZoneNameServers`, `PublicServiceUrl`, `LoadBalancerDnsName`.
