# Reaching the service privately (VPN-like access via SSM)

Some notes on private access to the deployed service via SSM: the task stays
fully private (`AssignPublicIp: DISABLED`, `IngressCidr` left as the VPC CIDR),
and you reach it through an SSM port-forwarding tunnel instead of any public
exposure — the same shape as a customer sitting behind a VPN into their VPC.

## What it needs beyond the base stack

1. **A small bastion EC2 instance** in the same VPC (e.g. `t3.micro`), with:
   - the SSM Agent (preinstalled on Amazon Linux 2/2023 AMIs)
   - an instance profile with the `AmazonSSMManagedInstanceCore` managed policy
   - no inbound security group rules needed — SSM sessions are agent-initiated
     outbound, so the bastion needs egress to SSM endpoints (either a public
     IP/NAT, or the `ssm`, `ssmmessages`, `ec2messages` VPC interface
     endpoints if fully private)
2. **A security group rule on `ServiceSecurityGroup`** allowing inbound on
   `ContainerPort` from the bastion's security group (instead of, or in
   addition to, `IngressCidr`).
3. **The task's private IP**, looked up per-run since it changes on every
   replacement:
   ```sh
   TASK_ARN=$(aws ecs list-tasks --cluster agents-cluster --service-name agent-server-dev --query 'taskArns[0]' --output text)
   ENI_ID=$(aws ecs describe-tasks --cluster agents-cluster --tasks "$TASK_ARN" \
     --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text)
   TASK_IP=$(aws ec2 describe-network-interfaces --network-interface-ids "$ENI_ID" \
     --query 'NetworkInterfaces[0].PrivateIpAddress' --output text)
   ```

## Starting the tunnel

Either from the CLI:

```sh
aws ssm start-session \
  --target <bastion-instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters "{\"host\":[\"$TASK_IP\"],\"portNumber\":[\"8000\"],\"localPortNumber\":[\"8000\"]}"
```

or entirely from the console: **EC2 → Instances → (bastion) → Connect →
Session Manager**, then start a port-forwarding session with the same
host/port. No CLI required on the reader's end, which is a reasonable stand-in
for "the customer clicks through their own tooling to get to the VPC."

Then open `http://localhost:8000/` in a real local browser — traffic tunnels
through SSM to the private task; nothing is ever exposed publicly.

## Don't forget the mixer task's own egress

This doc covers reaching the *bastion*. Separately, the `mixer` task itself
(with `AssignPublicIp: DISABLED`) needs a NAT gateway in its subnets to reach
`AuthPublicKeyEndpoint` (the JWKS endpoint, if used for auth) — that's an
external domain, not an AWS service, so the `ecr`/`logs`/`elasticfilesystem` VPC
endpoints that cover its other dependencies don't help here. See the main
`README.md`'s Prerequisites section. Skipping the NAT gateway won't stop the
task from starting; it'll fail the first time it tries to validate a token.

## Tradeoffs vs. plain single-IP public access

- More setup (one extra EC2 instance + IAM instance profile + one more SG
  rule) for a scenario that's genuinely closer to production.
- No stale-public-IP problem — you look up the task's current private IP
  each time instead of guessing whether last run's IP is still valid.
- Nothing about the service itself is exposed to the internet, matching the
  chart's original `ClusterIP`-only posture.

If/when this becomes the default demo path instead of a manual afterthought,
worth turning the bastion into its own optional CloudFormation resource
(behind a parameter/condition) rather than a manual setup step.
