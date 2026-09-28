# ecs-fargate-vpn-plus-alb

Runs the "mixer" server on ECS Fargate, reachable two ways at once: a
private **AWS Client VPN** path over TLS for normal use, plus a public
**Application Load Balancer** that exposes a small, explicit set of URL
paths (webhooks, an OAuth callback, etc.) to callers that can't be on the
VPN. The stack creates its own VPC from scratch — a public and private
subnet pair in each of two AZs, an Internet Gateway, a NAT Gateway, and the
route tables wiring it together — plus a public Route 53 hosted zone and
two DNS-validated ACM certificates for a subdomain you own, so the whole
thing is self-contained beyond delegating that subdomain's NS records at
your existing DNS provider and pointing a Client VPN endpoint at the VPC
this stack creates.

The stack needs its own VPC because the default VPC won't work here: an ALB
target group requires every target's AZ to be one the load balancer itself
has a subnet in, so running the task in private subnets while the ALB sits
in public ones needs at least two subnets per AZ (one public, one private).
The default VPC only has one subnet per AZ.

## Two ways to reach the service

- **Client VPN, over TLS** (`https://internal.${SubdomainName}`) — for
  Remix Desktop / interactive users. Full access to everything the app
  serves. See "TLS for Client VPN users" below for how this gets a real,
  publicly-trusted certificate despite the name never being reachable
  outside the VPC.
- **Public ALB** (`https://${SubdomainName}/...`) — for callers that are
  never going to be VPN-connected, like a webhook sender (e.g. Google
  Calendar push notifications) or an OAuth provider's redirect. Only the
  specific paths wired up as `AWS::ElasticLoadBalancingV2::ListenerRule`
  resources are reachable; everything else gets a flat 403 from the ALB's
  default action, without ever reaching a task. See `template.yaml`'s
  `HttpsListener` and the `APIDocsListenerRule`/`OauthCallbackListenerRule`/
  `OauthLoginListenerRule` examples — add one rule per real path you need,
  and don't change the listener's default action to a forward.

Both paths route to the same ECS service and the same task — the service is
registered with both `TargetGroup` (public ALB) and `InternalTargetGroup`
(internal ALB), since AWS doesn't allow one target group to be attached to
listeners on two different load balancers — and `mixer` differentiates by
URL path internally, same as any app with both public and private routes.

## TLS for Client VPN users

Reaching the task in plaintext isn't good enough: a client speaking plain
HTTP inside a VPC is no more secure than one speaking it over the open
internet, and it's exactly what makes strict TLS-only clients refuse the
connection. So Client VPN users terminate TLS on a dedicated internal ALB,
with a real, publicly-trusted certificate, instead of talking to the task
directly:

- **`InternalHostedZone`** — a **private** `AWS::Route53::HostedZone` named
  `internal.${SubdomainName}`, associated with this stack's own VPC. It's a
  sub-subdomain of the public name this stack manages, not a separate
  domain — and deliberately *not* the same name as `PublicHostedZone`: a
  private zone with an identical name would shadow *all* of the public
  zone's records for anything resolving inside the VPC (Client VPN clients
  included), not just the one name meant to be private.
- **`InternalCertificate`** — a normal, publicly-trusted ACM certificate for
  `internal.${SubdomainName}`, DNS-validated against **`PublicHostedZone`**
  (the public zone, not the private one — DNS validation only proves you
  control the name via the real, delegated DNS tree; it has nothing to do
  with where the name actually resolves for clients). Because it's a
  normal ACM certificate, no client-side trust configuration is needed —
  it's already trusted by anything that trusts the public CA system.
- **`InternalLoadBalancer`** — an **internal**-scheme ALB in the private
  subnets (`SubnetOne`/`SubnetTwo`), fronted by `InternalAlbSecurityGroup`.
  Its `InternalHttpsListener` terminates TLS with `InternalCertificate` and
  forwards to `InternalTargetGroup` — its own target group, not the public
  ALB's `TargetGroup`, since AWS rejects attaching one target group to two
  different load balancers. Both target groups point at the same task, on
  the same container and port; `Service.LoadBalancers` just lists both.
  Unlike the public ALB, there's no path restriction here — reaching this
  ALB at all already means being on the VPN. `InternalHttpListener` (port
  80) just redirects to 443, same pattern as the public ALB.
- **`InternalDnsRecord`** — an alias record in `InternalHostedZone` pointing
  `internal.${SubdomainName}` at `InternalLoadBalancer`. Route 53 Resolver
  answers this only for queries originating inside the VPC — including a
  Client VPN client using its pushed DNS resolver — and the name doesn't
  exist anywhere on the public internet.
- `ServiceSecurityGroup` only trusts `InternalAlbSecurityGroup` (Client VPN
  traffic, via the internal ALB) and `AlbSecurityGroup` (the public ALB) on
  `ContainerPort` — nothing else can reach the task directly, so there's no
  plaintext path to fall back to by accident.

One consequence worth knowing: ACM certificates are logged publicly in
Certificate Transparency logs, so `internal.${SubdomainName}` becomes
discoverable there even though it never resolves or routes anywhere for an
outside caller. That's an information leak (the hostname becomes visible),
not a reachability one — there's still nothing an outside caller can
connect to.

## What's in the VPC

- **`Vpc`** (default `10.0.0.0/16`) with DNS support and DNS hostnames both
  enabled — required for `InternalHostedZone` (and any private hosted zone)
  to resolve at all, and for Client VPN's pushed DNS resolver to work.
- **`PublicSubnetOne`/`PublicSubnetTwo`** (one per AZ) — routed to
  **`InternetGateway`** via `PublicRouteTable`. Host the public ALB, and the
  single **`NatGateway`** (in `PublicSubnetOne` only — cheaper for an
  example; a real deployment wanting per-AZ egress resilience would give
  each AZ its own NAT Gateway instead of sharing one across both).
- **`SubnetOne`/`SubnetTwo`** (one per AZ, matching the public pair's AZs) —
  routed to the NAT Gateway via `PrivateRouteTable`. Host the Fargate task,
  the EFS mount targets, and `InternalLoadBalancer`.
- **`ServiceSecurityGroup`** — the Fargate task. Only accepts `ContainerPort`
  traffic from `InternalAlbSecurityGroup` or `AlbSecurityGroup` (see above).
- **`AlbSecurityGroup`** — the public ALB. Open on 443/80 to the internet;
  80 only ever redirects to 443, never forwards to the task.
- **`InternalAlbSecurityGroup`** — the internal ALB. Trusts
  `SubnetOne`/`SubnetTwo`'s own CIDR blocks directly (`!GetAtt
  SubnetOne.CidrBlock`) rather than a separate parameter you'd have to
  discover and feed back in — since this stack defines those subnets
  itself, it already knows their CIDRs. Client VPN traffic arrives NATed to
  an address in whichever subnet the endpoint is associated with, so
  pointing the Client VPN endpoint's target-network association at these
  same two subnets (see `client-vpn-setup.md`) is all that's needed; there's
  nothing left to reconcile.
- **`EfsSecurityGroup`** — the EFS mount targets, reachable only from
  `ServiceSecurityGroup`.

The EFS volume and ECS task definition follow the same shape as
[`ecs-fargate-client-vpn`](../ecs-fargate-client-vpn/README.md) — see that
README if you want the reasoning behind the EFS access point, the task's
health check, or its IAM roles.

## IAM permissions

The admin IAM policy for deploying this stack is split into three files —
a single combined policy exceeds IAM's 6,144-character limit for a
customer-managed policy, so attach all three to the deploying principal;
none alone is sufficient:

- [`admin-iam-policy-core.json`](admin-iam-policy-core.json) — CloudFormation,
  task/execution IAM roles, EC2 describe actions and security groups, ECS,
  the private (`internal.${SubdomainName}`) hosted zone, EFS, logs.
- [`admin-iam-policy-networking.json`](admin-iam-policy-networking.json) —
  the VPC itself: subnets, route tables, the Internet Gateway, the NAT
  Gateway and its Elastic IP.
- [`admin-iam-policy-public-endpoint.json`](admin-iam-policy-public-endpoint.json) —
  the public hosted zone, ACM certificate lifecycle (covers both
  certificates), the ELB service-linked role, and the
  `elasticloadbalancing:*` actions for both ALBs' target group/listeners/
  rules.

Three grants in `admin-iam-policy-public-endpoint.json` cover actions this
template never calls directly, because the underlying AWS resource handler
calls them itself as part of its own create/update/delete flow:

- `route53:ListQueryLoggingConfigs` — `AWS::Route53::HostedZone`'s delete
  handler checks for a query-logging config to clean up first, and 403s
  without it even though this stack never configures one.
- `Ec2LookupsForAlbProvisioning` (`DescribeInternetGateways`,
  `DescribeAccountAttributes`, `DescribeAddresses`, `DescribeTags`) — what
  `CreateLoadBalancer` needs to validate the subnets you gave it actually
  route to an Internet Gateway.
- `elasticloadbalancing:SetRulePriorities` — `AWS::ElasticLoadBalancingV2::ListenerRule`'s
  update handler calls this itself to shuffle priorities out of the way
  before it can create/replace a rule, any time a stack update adds,
  removes, or reorders listener rules.

## The Client VPN endpoint comes after this stack, not before

Because this stack creates its own VPC, the Client VPN endpoint has to be
created (or re-pointed) afterward, targeting this stack's own `VpcId`/
`SubnetOneId`/`SubnetTwoId` Outputs — there's no VPC to point at until the
stack exists. [`client-vpn-setup.md`](client-vpn-setup.md) walks through
standing one up in a test account, starting from those Outputs.

## Deploying

Stack creation has one real chicken-and-egg: the hosted zone doesn't exist
until the stack creates it, but the certificates won't validate until that
zone's NS records are delegated. The VPC/subnet/routing resources don't
have this problem — they're ordinary, fast-creating resources with no
external dependency, so they're done long before the certificates are the
thing you're waiting on.

```sh
aws cloudformation create-stack \
  --stack-name mixer-dev \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters file://parameters.example.json
```

This call returns immediately, but the stack itself will sit at
`CREATE_IN_PROGRESS` for a while — `PublicHostedZone` and
`InternalHostedZone` are both created almost instantly, but `Certificate`
and `InternalCertificate` both block until ACM can see the DNS validation
records it wrote into `PublicHostedZone` (both certificates validate
against the public zone — see "TLS for Client VPN users" for why
`InternalCertificate`, despite being for a private-only name, still
validates there), which can't happen until the zone is actually delegated.
While it's sitting there:

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
   `Certificate`/`InternalCertificate` resources during ordinary
   propagation delay — you don't need to re-run anything.
4. Once both certificates validate, the rest of the stack (both ALBs,
   listeners, rules, both alias records) creates normally and the stack
   reaches `CREATE_COMPLETE`.

If you'd rather not straddle a long-running `create-stack` call, you can
split this into two runs instead: first deploy with only `PublicHostedZone`
and `InternalHostedZone` reachable (e.g. comment out everything from
`Certificate`/`InternalCertificate` on down), fetch the public zone's name
servers, delegate them, confirm delegation with `dig NS $SUBDOMAIN`, then
uncomment the rest and `update-stack`. The single-call version above is
simpler when you don't mind the wait.

Once `CREATE_COMPLETE`, follow [`client-vpn-setup.md`](client-vpn-setup.md)
to stand up the Client VPN endpoint against this stack's own VPC.

## Persistent signing key

Same as the base example: generate the key into Secrets Manager and pass the
secret's ARN as the optional `ServerKeySecretArn` parameter. See
[Persistent signing key](../ecs-fargate/README.md#persistent-signing-key)
in the `ecs-fargate` README for the command and the details.

## Reaching the service

- **Via VPN**: connect to the Client VPN (see `client-vpn-setup.md`),
  browse `https://internal.${SubdomainName}/` — a normal, publicly-trusted
  TLS connection, port 443. No custom CA or client trust configuration
  needed.
- **Via the public ALB**, only for the specific paths wired up:
  ```sh
  curl https://<SubdomainName>/webhooks/whatever
  ```
  Anything else (`curl https://<SubdomainName>/`, for instance) gets a 403.

## Adding a new exposed public path

Add another `AWS::ElasticLoadBalancingV2::ListenerRule` pointed at
`HttpsListener`, with its own `Priority` (rules are evaluated in ascending
priority order, first match wins) and a `path-pattern` condition, forwarding
to the existing `TargetGroup`. Don't add a broad pattern like `/*` — that
defeats the whole reason for using path-based rules instead of just exposing
the ALB's default action. (There's no equivalent restriction on
`InternalHttpsListener` — anything reaching it is already a VPN client with
full access.)

## Deleting the stack

As in the base example, the EFS filesystem is retained when the stack is
deleted, and has daily automatic backups turned on. See
[Deleting the stack](../ecs-fargate/README.md#deleting-the-stack) in the
`ecs-fargate` README.

## Outputs

`ClusterName`, `ServiceName`, `TaskDefinitionArn`, `EfsFileSystemId`,
`EfsAccessPointId`, `LogGroupName`, plus `VpcId`, `SubnetOneId`,
`SubnetTwoId`, `PublicSubnetOneId`, `PublicSubnetTwoId` (needed for
`client-vpn-setup.md`), `PublicHostedZoneId`, `PublicHostedZoneNameServers`,
`PublicServiceUrl`, `LoadBalancerDnsName`, `InternalServiceUrl` (what VPN
clients should actually connect to), `InternalHostedZoneId`,
`InternalLoadBalancerDnsName`.
