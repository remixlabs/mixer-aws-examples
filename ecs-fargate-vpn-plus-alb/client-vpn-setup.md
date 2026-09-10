# Setting up AWS Client VPN from scratch (test account)

This walks through standing up a minimal AWS Client VPN endpoint against the
VPC that `template.yaml` in this directory creates for itself. It uses
mutual TLS certificate authentication with a self-signed test CA — the
standard AWS quick-start path — which is fine for a disposable test account
but **not** how a real customer would run this: a production setup would
use their existing corporate PKI or federated SAML auth
(`AWS::EC2::ClientVpnEndpoint`'s `federated-authentication` type) against
their IdP instead of hand-rolled certificates.

**Run this after deploying the mixer stack, not before.** Unlike a template
that assumes a VPC already exists, this one creates its own — so the
`VpcId`/`SubnetOneId`/`SubnetTwoId` this file needs come from that stack's
Outputs, not from something you already had lying around:

```sh
VPC_ID=$(aws cloudformation describe-stacks --stack-name mixer-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)
SUBNET_ONE=$(aws cloudformation describe-stacks --stack-name mixer-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`SubnetOneId`].OutputValue' --output text)
SUBNET_TWO=$(aws cloudformation describe-stacks --stack-name mixer-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`SubnetTwoId`].OutputValue' --output text)
```

This is a one-time, admin-run setup — a different persona and a different,
broader set of permissions than `admin-iam-policy-core.json` +
`admin-iam-policy-public-endpoint.json` + `admin-iam-policy-networking.json`
in this directory (which only cover day-to-day deploys of the mixer stack
itself).

## What you need beyond the deployed mixer stack

- `aws` CLI configured against the test account.
- `easy-rsa` (or any tool that gets you an X.509 CA + server + client cert)
  to generate certificates.
- An IAM identity with at least the permissions in
  [`client-vpn-setup-iam-policy.json`](client-vpn-setup-iam-policy.json) —
  ACM cert import/delete plus the Client VPN endpoint lifecycle actions used
  below. For a disposable test account, `AdministratorAccess` is simplest,
  same reasoning as the base example's Prerequisites section.

## 1. Generate a test CA, server cert, and client cert

```sh
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
./easyrsa init-pki
./easyrsa build-ca nopass                          # -> pki/ca.crt, pki/private/ca.key
./easyrsa build-server-full server.mixer-vpn.test nopass
  # -> pki/issued/server.mixer-vpn.test.crt, pki/private/server.mixer-vpn.test.key
./easyrsa build-client-full mixer-test-client nopass
  # -> pki/issued/mixer-test-client.crt, pki/private/mixer-test-client.key
```

`nopass` is fine for a throwaway test CA; don't reuse this CA or its
unencrypted key for anything real.

The server cert's short name has to look like a domain (contain a dot) —
Client VPN validates the server certificate like a normal TLS server cert,
and rejects a bare CN like `server` with `Certificate ... does not have a
domain`. It doesn't need to resolve to anything real, since nothing ever
looks it up by DNS here; `server.mixer-vpn.test` just needs to be
FQDN-shaped. The client cert has no such requirement.

## 2. Import certificates into ACM

Client VPN reads its server and CA certs from ACM, not from raw files.

```sh
SERVER_CERT_ARN=$(aws acm import-certificate \
  --certificate fileb://pki/issued/server.mixer-vpn.test.crt \
  --private-key fileb://pki/private/server.mixer-vpn.test.key \
  --certificate-chain fileb://pki/ca.crt \
  --query CertificateArn --output text)

# The "client root certificate chain" ACM entry is what Client VPN validates
# connecting clients against. Import the CA's own cert+key as its own ACM
# certificate for this — acceptable for a disposable test CA, never for a
# CA you actually care about, since ACM now holds its private key.
CA_CERT_ARN=$(aws acm import-certificate \
  --certificate fileb://pki/ca.crt \
  --private-key fileb://pki/private/ca.key \
  --query CertificateArn --output text)
```

## 3. Create the Client VPN endpoint

```sh
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query 'Vpcs[0].CidrBlock' --output text)
# .2 of the VPC's own CIDR is always the reserved Amazon-provided DNS
# resolver address. Passing it as --dns-servers is what lets connected
# clients resolve private DNS names in this VPC — e.g. the Cloud Map/Route 53
# private hosted zone template.yaml creates for the mixer service (see the
# README's "Reaching the service"). Without it, each client just keeps
# using its own pre-existing DNS server, which has no idea these private
# names exist.
RESOLVER_IP=$(python3 -c "import ipaddress,sys; print(ipaddress.ip_network(sys.argv[1]).network_address + 2)" "$VPC_CIDR")

CLIENT_VPN_ID=$(aws ec2 create-client-vpn-endpoint \
  --description "mixer test Client VPN" \
  --client-cidr-block 10.100.0.0/22 \
  --server-certificate-arn "$SERVER_CERT_ARN" \
  --authentication-options "Type=certificate-authentication,MutualAuthentication={ClientRootCertificateChainArn=$CA_CERT_ARN}" \
  --connection-log-options Enabled=false \
  --split-tunnel \
  --dns-servers "$RESOLVER_IP" \
  --vpc-id "$VPC_ID" \
  --query ClientVpnEndpointId --output text)
```

The resolver IP falls inside `VPC_CIDR`, which step 5 authorizes, so no
extra authorization rule is needed for it. `10.100.0.0/22` (the VPN
client CIDR) is deliberately outside `VpcCidr` (`10.0.0.0/16` by default) —
Client VPN requires the two not to overlap.

`--split-tunnel` matters for a demo: without it, *all* client traffic
(including normal internet browsing) routes through the endpoint, which
needs its own internet egress and will otherwise strand the tester. With
split-tunnel, only traffic to authorized target networks (step 5) goes
through the VPN — everything else goes direct, same as before connecting.

`--connection-log-options Enabled=false` skips CloudWatch connection
logging, fine for a test endpoint; a real deployment would enable it.

On a genuinely fresh account, this first `create-client-vpn-endpoint` call
also creates the `AWSServiceRoleForClientVPN` service-linked role, using the
caller's own permissions — without `iam:CreateServiceLinkedRole` (included in
`client-vpn-setup-iam-policy.json`), it fails with `UnauthorizedOperation`
even though the role creation itself isn't something you asked for directly.
It's a one-time cost per account; harmless to leave granted after.

## 4. Associate target subnets

Associate with `$SUBNET_ONE`/`$SUBNET_TWO` — the mixer stack's own private
subnets, from its `SubnetOneId`/`SubnetTwoId` Outputs. This is also what
`ServiceSecurityGroup` in `template.yaml` already trusts by CIDR block
directly (see its ingress rule descriptions), so associating here is all
that's needed — there's no separate parameter to go back and set
afterward, unlike the plain `ecs-fargate-client-vpn` example where the VPC
already existed and the CIDRs had to be looked up and fed back in:

```sh
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" --subnet-id "$SUBNET_ONE"
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" --subnet-id "$SUBNET_TWO"
```

Each association takes a few minutes to go from `associating` to
`associated` (`aws ec2 describe-client-vpn-target-networks
--client-vpn-endpoint-id "$CLIENT_VPN_ID"`).

## 5. Authorize access to the VPC

Associating a subnet makes it *routable*; an authorization rule is what
actually lets connected clients reach it. Reuse `$VPC_CIDR` from step 3:

```sh
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" \
  --target-network-cidr "$VPC_CIDR" \
  --authorize-all-groups
```

(Authorizing the whole VPC CIDR is the simplest thing for a test endpoint;
`--target-network-cidr` can be narrowed to just `$SUBNET_ONE`/`$SUBNET_TWO`
if you want the authorization scoped tighter than "reach anything in the
VPC" — e.g. to exclude the public subnets the ALB/NAT Gateway sit in, which
VPN clients have no reason to reach directly.)

## 6. Export and finish the client config

```sh
aws ec2 export-client-vpn-client-configuration \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" --output text > mixer-test-client.ovpn
```

The exported `.ovpn` references cert/key files by path rather than embedding
them, which doesn't travel well. Append the client cert and key inline so
the profile is self-contained:

```sh
{
  echo "<cert>"
  cat pki/issued/mixer-test-client.crt
  echo "</cert>"
  echo "<key>"
  cat pki/private/mixer-test-client.key
  echo "</key>"
} >> mixer-test-client.ovpn
```

Import `mixer-test-client.ovpn` into the [AWS-provided VPN
client](https://aws.amazon.com/vpn/client-vpn-download/) (or Tunnelblick /
OpenVPN Connect — it's a standard OpenVPN profile) and connect.

## 7. Verify

With the VPN connected, browse `http://mixer.mixer-${Env}.internal:8000/` —
see the README's "Reaching the service" section. Nothing left to configure
on the mixer stack's side: the security group already trusts
`$SUBNET_ONE`/`$SUBNET_TWO`'s own CIDR blocks (it's the same VPC the stack
created them in), and the NAT Gateway that gives the task its own egress
for ECR/logs/EFS was created by the stack too, not something to set up
separately here.

## Troubleshooting

**DNS resolution fails** (`http://mixer.mixer-${Env}.internal:8000/` gives
"could not resolve host", but the task is reachable by raw IP) even though
the endpoint was created with `--dns-servers "$RESOLVER_IP"` in step 3 — the
most common cause on macOS is another VPN-like tool (Tailscale with
MagicDNS is a common one) claiming the machine's global/unscoped DNS
resolver slot, so the Client VPN's pushed DNS server never gets consulted.
See [`../ecs-fargate-client-vpn/client-vpn-setup.md`](../ecs-fargate-client-vpn/client-vpn-setup.md)'s
"Troubleshooting" section for the full explanation and the `scutil --dns` /
`/etc/resolver` fix — it applies here unchanged, just substitute this
stack's own `$RESOLVER_IP` (`10.0.0.2` for the default `VpcCidr`,
`10.0.0.0/16`) and `mixer-${Env}.internal` domain.

**Connection hangs rather than being actively refused**: double check that
`$SUBNET_ONE`/`$SUBNET_TWO` (step 4, this stack's own `SubnetOneId`/
`SubnetTwoId` Outputs) are the subnets actually associated with the Client
VPN endpoint — not the VPN's own client CIDR block from step 3.

## Tearing down

```sh
aws ec2 disassociate-client-vpn-target-network --client-vpn-endpoint-id "$CLIENT_VPN_ID" --association-id <assoc-id>  # once per association
aws ec2 delete-client-vpn-endpoint --client-vpn-endpoint-id "$CLIENT_VPN_ID"
aws acm delete-certificate --certificate-arn "$SERVER_CERT_ARN"
aws acm delete-certificate --certificate-arn "$CA_CERT_ARN"
```

Client VPN bills per association-hour and per connection-hour even when
idle, so don't leave a test endpoint associated longer than you need it.
Do this before deleting the mixer stack itself — CloudFormation can't
delete the VPC while a Client VPN endpoint is still associated with one of
its subnets.
