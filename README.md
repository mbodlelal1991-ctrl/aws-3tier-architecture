Internet
                            │
                    ┌───────▼────────┐
                    │  Internet GW   │
                    └───────┬────────┘
                            │
          ┌─────────────────┴─────────────────┐
          │         Public Subnets            │
          │   10.0.0.0/24    10.0.1.0/24      │
          │   (us-east-1a)   (us-east-1b)     │
          │                                    │
          │    ┌──────────────────────┐       │
          │    │  Application LB       │       │
          │    │  (ALB-SG: 80 open)    │       │
          │    └──────────┬────────────┘       │
          │               │                     │
          │    ┌──────────▼────────────┐       │
          │    │   NAT Gateway          │       │
          │    │  (outbound only)       │       │
          │    └──────────┬────────────┘       │
          └───────────────┼────────────────────┘
                           │
          ┌────────────────┼────────────────────┐
          │         Private App Subnets          │
          │   10.0.10.0/24     10.0.11.0/24     │
          │   (us-east-1a)     (us-east-1b)     │
          │                                       │
          │   ┌─────────┐        ┌─────────┐    │
          │   │  EC2     │        │  EC2     │    │
          │   │ (ASG,    │        │ (ASG,    │    │
          │   │  nginx)  │        │  nginx)  │    │
          │   │ App-SG   │        │ App-SG   │    │
          │   └────┬────┘        └────┬────┘    │
          └────────┼──────────────────┼──────────┘
                    │                  │
          ┌─────────┼──────────────────┼──────────┐
          │         Private DB Subnets            │
          │   10.0.20.0/24     10.0.21.0/24      │               
          │   (us-east-1a)     (us-east-1b)      │
          │                                        │
          │        ┌─────────────────┐            │
          │        │  RDS PostgreSQL  │            │
          │        │     DB-SG        │            │
          │        └─────────────────┘            │
          └────────────────────────────────────────┘
          ## Components

| Layer | Service | Configuration |
|---|---|---|
| Network | VPC | `10.0.0.0/16`, 6 subnets across 2 AZs (public / app / db tiers) |
| Compute | EC2 Auto Scaling Group | `t3.micro`, min 1 / desired 2 / max 3, launched via Launch Template |
| Load Balancing | Application Load Balancer | Internet-facing, HTTP:80, spans both public subnets |
| Database | RDS PostgreSQL | `db.t3.micro`, private subnets, single-AZ (cost-optimized for demo) |
| Outbound Access | NAT Gateway | Single-AZ, enables private-subnet package installs (`dnf`) |
| Security | 3-tier Security Group chain | ALB-SG → App-SG → DB-SG, each scoped to reference the SG in front of it rather than open CIDRs |

## Security Design

Traffic can only flow one direction through the stack:

- **ALB-SG**: allows inbound `80` from `0.0.0.0/0` (public entry point)
- **App-SG**: allows inbound `80` **only from ALB-SG** — no direct public access to app instances
- **DB-SG**: allows inbound `5432` **only from App-SG**, with outbound rules removed entirely (RDS never initiates outbound connections)

This means even if an attacker discovered an EC2 instance's private IP, they couldn't reach it — App-SG only trusts traffic that's already passed through the ALB.

## Key Design Decisions

- **No NAT Gateway in the original design** — initially skipped to preserve Free Tier costs. Re-added after discovering private-subnet instances need outbound internet access for OS package installs (`dnf install nginx`) even when running fully private application logic. This is a common real-world gap between "textbook private subnet" and "instances still need to patch/install software."
- **Private DB subnet group spans 2 AZs** even though RDS itself is single-AZ, because AWS requires a subnet group with 2+ AZs regardless of Multi-AZ deployment status.
- **Security groups reference each other by ID, not CIDR** — the standard least-privilege pattern for internal AWS traffic.

## CI/CD Pipeline

Once the base infrastructure was live, I added a GitHub Actions pipeline so app changes deploy automatically instead of manually terminating instances through the console:

**How it works:**
1. EC2 instances pull `app/index.html` live from this repo's `main` branch at boot time (via `curl` in the Launch Template's user data), instead of having content hardcoded into the template.
2. A push to `app/**` on `main` triggers `.github/workflows/main.yml`, which:
   - Authenticates to AWS using an IAM user (`github-actions-deploy`) scoped to only the EC2/ASG permissions it needs — no broad admin access, credentials stored as GitHub encrypted secrets, never in code.
   - Creates a new Launch Template version and sets it as default.
   - Triggers an Auto Scaling Group **Instance Refresh**, which gradually replaces running instances so each one boots with the current template and pulls the latest `index.html`.
3. Result: editing `app/index.html` and pushing to `main` updates the live site with no manual AWS console steps.

**Known limitation:** deploys take a few minutes because they go through full instance replacement rather than an in-place update — a more advanced setup would serve static assets from S3/CloudFront or use containers for near-instant deploys. Documenting the trade-off here rather than treating it as a gap.

## What I'd Add With More Time / Budget

- Multi-AZ RDS for real failover (not Free Tier eligible)
- HTTPS via ACM certificate + Route 53 custom domain
- Infrastructure as Code (Terraform/CloudFormation) instead of manual console setup for the base resources
- CloudWatch alarms + SNS notifications for ASG scaling events and RDS metrics
- S3 + CloudFront for static asset delivery, decoupled from instance boot time

## Troubleshooting Log (Real Issues Hit During This Build)

**Infrastructure:**
1. **ALB timing out** — root cause was an empty ALB-SG with zero inbound rules; fixed by adding HTTP:80 from `0.0.0.0/0`.
2. **502 Bad Gateway from ALB** — target health checks failing because EC2 instances in private subnets had no outbound internet route, so `dnf install nginx` in the user data script silently failed. Fixed by adding a NAT Gateway + updating the private route table.
3. **Confirmed via target group health status** (`0 healthy / 1 unhealthy`) before and after each fix — a good habit for isolating whether a problem is at the ALB, security group, or instance level.

**CI/CD pipeline:**
4. **Region validation error in GitHub Actions** — the `AWS_REGION` secret had hidden characters from a copy-paste; fixed by retyping the value manually instead of pasting.
5. **Malformed AWS credentials** — a corrupted Secret Access Key produced an opaque "Invalid key=value pair" authorization error. Resolved by rotating the IAM access key entirely rather than continuing to debug a secret that may have been compromised or logged in cleartext during troubleshooting — the safer call when credentials misbehave.
6. **Workflow editing the wrong file** — GitHub Actions only recognizes workflows under the dot-prefixed `.github/workflows/` directory. An early edit went to a decoy `github/workflows/` path (missing the leading dot) and silently had no effect, wasting a debugging cycle before I caught it by checking the file's full breadcrumb path.
7. **AWS CLI `ParamValidationError: --launch-template-data required`** — my own workflow bug; `create-launch-template-version` requires this argument even when cloning the source version unchanged. Fixed by passing `--launch-template-data '{}'`.
8. **`AutoScalingGroup name not found`** — ASG names are case-sensitive; the workflow was written with lowercase `app-asg` but the actual resource was `App-ASG`. Fixed by matching the exact console name.

The pattern across most of these: verify the exact resource name/value in the AWS console rather than assuming it matches what was typed earlier, and isolate failures one pipeline step at a time using the Actions log rather than guessing at the whole workflow.

---
*Built as a portfolio project to demonstrate AWS Solutions Architect Associate (SAA-C03) concepts in a working environment.*
