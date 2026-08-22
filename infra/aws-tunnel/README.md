# AWS rendezvous host

Replaces the Cloudflare quick tunnel with a small EC2 host you control.
The Mac dials **out** to it over a reverse SSH tunnel (`ssh -R`) - no
inbound port is ever opened on the Mac or the home router - and a phone
browser reaches it over HTTPS. The relay itself never leaves loopback on
the Mac.

```
Phone browser ──HTTPS──▶ Caddy :443 ──▶ 127.0.0.1:9375 (EC2)
                                              ▲
                                              │ ssh -R 127.0.0.1:9375:127.0.0.1:8375
                                              │ (Mac dials out, autossh-supervised)
                                              │
                                    relay :8375 (127.0.0.1 only, on the Mac)
```

## Why CloudFormation, not CDK

`landvera/infra` uses CDK, but that's a multi-stack application with
shared constructs and a real deploy pipeline. This is one instance, one
security group, one Elastic IP, and an optional DNS record - a use case
CloudFormation covers directly, in a single file a reviewer can read
top to bottom without an `npm install` or a `cdk synth`. That fits this
repo's existing style (`web/index.html` is one file with no build step)
better than standing up a second CDK app for four resources. If this
grows more stacks or more logic, revisit.

## What this does NOT do

This PR ships the template only. Nothing is applied, no instance is
launched, no hostname is registered. That is a deliberate, separate step
the captain runs by hand (see below) - not something this change does
for you.

## Prerequisites

- AWS CLI configured with the `landerafs` profile against account
  `450730497623`, region `us-east-1`.
- A domain you control, in whatever DNS provider you like. Route 53 is
  optional (see `HostedZoneId` below) - the template works with any DNS
  host, since it only needs an A record pointing at the Elastic IP.
- An SSH keypair generated **just for this tunnel** - do not reuse an
  existing identity:
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/herdr-remote-tunnel -C herdr-remote-tunnel -N ""
  ```
  The private key never leaves the Mac and never enters this repo. Only
  the `.pub` file's contents go into the stack, as the `TunnelPublicKey`
  parameter.
- Your current public IP, for `SshAllowedCidr` (`curl -s ifconfig.me`).

## Deploy (run by a human, not by this PR)

```bash
aws cloudformation deploy \
  --profile landerafs --region us-east-1 \
  --stack-name herdr-remote-tunnel \
  --template-file infra/aws-tunnel/cloudformation.yaml \
  --parameter-overrides \
      HostnameFqdn=herdr-remote.example.com \
      TunnelPublicKey="$(cat ~/.ssh/herdr-remote-tunnel.pub)" \
      SshAllowedCidr="$(curl -s ifconfig.me)/32" \
  --capabilities CAPABILITY_IAM
```

Add `HostedZoneId=Z0123456789ABC` to the overrides if the domain's
hosted zone lives in this same account and you want the template to
create the A record for you. Otherwise, after deploy:

```bash
aws cloudformation describe-stacks --profile landerafs --region us-east-1 \
  --stack-name herdr-remote-tunnel \
  --query "Stacks[0].Outputs"
```

and create an A record for `HostnameFqdn` pointing at the printed
`ElasticIp` in whatever DNS provider hosts that domain.

DNS must resolve and Caddy must have obtained its certificate (usually
under a minute after the A record propagates) before the Mac's tunnel or
a phone browser can reach it over HTTPS.

## Updating the SSH-allowed CIDR

Home IPs on residential ISPs are not always static. If `SshAllowedCidr`
goes stale, the reverse tunnel will fail to connect. Update the security
group directly rather than re-running the whole stack:

```bash
aws ec2 update-security-group-rule-descriptions-ingress ...  # or:
aws cloudformation deploy ... --parameter-overrides SshAllowedCidr="$(curl -s ifconfig.me)/32" ...
```

The second form is simplest - CloudFormation only touches the changed
rule.

## Cost

All figures are `us-east-1` on-demand, August 2026 pricing, for the
resources this template creates:

| Resource | Cost |
|---|---|
| `t4g.nano` (2 vCPU burstable, 0.5 GiB), running 24/7 | ~$3.00/mo |
| Elastic IP, attached to a running instance | $0.00/mo (free while associated) |
| EBS gp3 8 GiB root volume | ~$0.65/mo |
| Data transfer out (phone/browser traffic; tiny for this use case) | ~$0.10-0.50/mo |
| Route 53 hosted zone (only if you opt in via `HostedZoneId` and don't already have one) | $0.50/mo |

**Total: roughly $4/mo**, or ~$4.50/mo if you also stand up a fresh
Route 53 zone for this. No per-request or bandwidth surprises at this
traffic level - this is a single phone dashboard polling one relay, not
a public service.

## Security model, in one place

- The relay stays bound to `127.0.0.1` on the Mac. This stack cannot
  reach it directly - only the tunnel the Mac opens can.
- SSH accepts exactly one identity: the forwarding-only `herdr-tunnel`
  user, restricted with the `restrict,port-forwarding` authorized_keys
  option (no shell, no command execution, no agent/X11 forwarding -
  just the `-R` forward this exists for).
- `GatewayPorts no` (the default, set explicitly) means the tunnel's
  remote port binds to the EC2 host's loopback only - nothing but Caddy,
  running on that same host, can reach it.
- The security group opens only 443 (public) and 22 (restricted to
  `SshAllowedCidr`). Port 80 is never opened; Caddy issues its
  Let's-Encrypt certificate over TLS-ALPN-01 on 443 alone.
- Admin access to the host itself is via AWS Systems Manager Session
  Manager (`aws ssm start-session`), not a general SSH login - the IAM
  instance profile grants only `AmazonSSMManagedInstanceCore`.
- `HERDR_RELAY_TOKEN` is unaffected by any of this and stays required at
  the relay - this stack is transport only, not an auth boundary.
