# BEM12 Lab 1 — Auto Scaling Web Tier (CloudFormation)

This repository contains a single CloudFormation template,
`autoscaling-web-tier.yaml`, that provisions a highly available, auto-scaling
web tier on AWS. It satisfies the customer requirements of: a single public
endpoint, automatic scaling on CPU spikes, even traffic distribution, no public
exposure of the backend servers, and full Infrastructure-as-Code repeatability.

## Architecture at a glance

```
                  ┌──────────────────────────────────────┐
   Internet ─────►│  Application Load Balancer (public)  │
                  └─────────────┬────────────────────────┘
                                │ HTTP :80 (ALB SG only)
                ┌───────────────┴───────────────┐
                ▼                               ▼
       ┌────────────────┐              ┌────────────────┐
       │ EC2 (private)  │              │ EC2 (private)  │  ◄── Auto Scaling Group
       │   AZ 1         │              │   AZ 2         │      target-tracks CPU %
       └────────────────┘              └────────────────┘
                │                               │
                └────────────► NAT Gateway ─────► Internet (egress only)
```

Key components: VPC across **2 Availability Zones**, public subnets for the
ALB + NAT Gateway, private subnets for the EC2 fleet, an Auto Scaling Group
behind an Application Load Balancer, target-tracking scaling on CPU, IMDSv2,
and SSM Session Manager (no SSH) for any administration.

## Mapping to the customer requirements

| Requirement | Where it lives in the template |
|---|---|
| Single public endpoint | `ApplicationLoadBalancer` + `ApplicationUrl` output |
| Even traffic distribution | `WebTargetGroup` + ALB round-robin |
| Auto-scale on CPU spikes | `CpuTargetTrackingPolicy` (target = `CpuTargetUtilization` %) |
| Servers in a secure private network | EC2 in `PrivateSubnet1/2`; `InstanceSecurityGroup` only allows traffic from the ALB SG; no public IPs |
| Each instance uniquely identifiable | `UserData` renders an HTML page with instance ID, AZ, private IP and hostname |
| Repeatable provisioning (IaC) | This single CloudFormation template, fully parameterised |
| Demonstrable scaling | `stress-ng` is pre-installed on every instance |

## Deploy

### Option A — CloudFormation Git sync (preferred)

This repository contains a Git sync deployment file at
`deployments/bem12-autoscaling-web.yaml`. After connecting the repository to
CloudFormation Git sync (Console → CloudFormation → **Git sync** → Create
sync), CloudFormation will read that file from the configured branch and
create / update the stack automatically every time you push a change. No
local AWS CLI calls are needed.

The deployment file pins the customer-required values:

```yaml
parameters:
  MinSize: "1"
  DesiredCapacity: "1"
  MaxSize: "4"
  CpuTargetUtilization: "30"
```

### Option B — One-off CLI deploy

```bash
aws cloudformation deploy \
  --stack-name bem12-autoscaling-web \
  --template-file autoscaling-web-tier.yaml \
  --capabilities CAPABILITY_IAM
```

When the stack finishes, the public URL is the `ApplicationUrl` output:

```bash
aws cloudformation describe-stacks \
  --stack-name bem12-autoscaling-web \
  --query "Stacks[0].Outputs[?OutputKey=='ApplicationUrl'].OutputValue" \
  --output text
```

Open that URL in a browser and refresh several times — the page reports the
instance ID, AZ and private IP serving the response, so you can visually
confirm requests are landing on different backends.

## Demonstrate scaling to stakeholders

1. Open the `ApplicationUrl` in a browser; note the current instance count
   in the EC2 → **Auto Scaling Groups** console.
2. From the AWS console, **Connect → Session Manager** into one of the
   instances (no SSH keys, no bastion host required).
3. Drive CPU above the target with the pre-installed tool:

   ```bash
   sudo stress-ng --cpu 2 --timeout 600s
   ```

4. Within ~2–3 minutes the target-tracking policy will launch additional
   instances. Refresh the browser — new instance IDs will start appearing in
   responses as the new instances pass health checks and join the target
   group.
5. Stop the stress test; the ASG will scale back down to `MinSize` once CPU
   drops, after the policy's cooldown.

## Clean up

```bash
aws cloudformation delete-stack --stack-name bem12-autoscaling-web
```

## Parameters worth knowing

| Parameter | Default | Notes |
|---|---|---|
| `InstanceType` | `t3.micro` | Burstable types are sufficient for this lab. |
| `LatestAmiId` | SSM-resolved Amazon Linux 2023 | Always picks the newest AL2023 AMI in the region. |
| `MinSize` / `DesiredCapacity` / `MaxSize` | 1 / 1 / 4 | Per spec: starts at 1 instance, scales out to at most 4. |
| `CpuTargetUtilization` | 30 | Per spec: ALB-fronted ASG scales out when average CPU > 30%. |
