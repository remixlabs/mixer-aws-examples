# Setting up AWS Client VPN from scratch (test account)

This walks through standing up a minimal AWS Client VPN endpoint against an
existing VPC, purely so `template.yaml` in this directory has something real
to assume already exists. It uses mutual TLS certificate authentication with
a self-signed test CA — the standard AWS quick-start path — which is fine for
a disposable test account but **not** how a real customer would run this: a
production setup would use their existing corporate PKI or federated SAML
auth (`AWS::EC2::ClientVpnEndpoint`'s `federated-authentication` type)
against their IdP instead of hand-rolled certificates.

This is a one-time, admin-run setup — a different persona and a different,
broader set of permissions than `admin-iam-policy.json` in this directory
(which only covers day-to-day deploys of the mixer stack against a VPN that
already exists).

## What you need beyond a VPC

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
# base README's "Reaching the service"). Without it, each client just keeps
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
extra authorization rule is needed for it.

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

Associate at least one subnet per AZ you want to route through — this is
also what auto-adds a route for the VPC's local CIDR to the endpoint's own
route table for that AZ:

```sh
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" --subnet-id "$SUBNET_ONE"
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" --subnet-id "$SUBNET_TWO"
```

Each association takes a few minutes to go from `associating` to
`associated` (`aws ec2 describe-client-vpn-target-networks
--client-vpn-endpoint-id "$CLIENT_VPN_ID"`).

Note the CIDRs of `$SUBNET_ONE`/`$SUBNET_TWO` themselves (`aws ec2
describe-subnets --subnet-ids "$SUBNET_ONE" "$SUBNET_TWO"`) — you'll need
them for `ClientVpnTargetSubnetCidrOne`/`Two` when deploying the stack, in
step 7 (that's the CIDR range mixer's security group needs to trust, not
the VPN's client CIDR block from step 3).

## 5. Authorize access to the VPC

Associating a subnet makes it *routable*; an authorization rule is what
actually lets connected clients reach it. Reuse `$VPC_CIDR` from step 3
(look it up fresh with `aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query
'Vpcs[0].CidrBlock' --output text` if you're running this step in a new
shell — a non-default VPC's CIDR isn't always `10.0.0.0/16`):

```sh
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id "$CLIENT_VPN_ID" \
  --target-network-cidr "$VPC_CIDR" \
  --authorize-all-groups
```

(Authorizing the whole VPC CIDR is the simplest thing for a test endpoint;
`--target-network-cidr` can be narrowed to just the mixer task's subnets if
you want the authorization scoped tighter than "reach anything in the VPC".)

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

With the VPN connected, set `ClientVpnTargetSubnetCidrOne`/`Two` in
`parameters.example.json` to the CIDRs of `$SUBNET_ONE`/`$SUBNET_TWO` from
step 4 — **not** `--client-cidr-block` from step 3 (traffic reaching the
VPC arrives NAT'd to an address in the associated subnets, not the client's
own address). Then follow the README to deploy the stack and reach the
task at its stable DNS name.

## Tearing down

```sh
aws ec2 disassociate-client-vpn-target-network --client-vpn-endpoint-id "$CLIENT_VPN_ID" --association-id <assoc-id>  # once per association
aws ec2 delete-client-vpn-endpoint --client-vpn-endpoint-id "$CLIENT_VPN_ID"
aws acm delete-certificate --certificate-arn "$SERVER_CERT_ARN"
aws acm delete-certificate --certificate-arn "$CA_CERT_ARN"
```

Client VPN bills per association-hour and per connection-hour even when
idle, so don't leave a test endpoint associated longer than you need it.
