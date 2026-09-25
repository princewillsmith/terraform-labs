# Terraform Labs

![terraform](https://github.com/princewillsmith/terraform-labs/actions/workflows/terraform.yml/badge.svg)

Infrastructure as Code labs for AWS networking. Every change is checked by GitHub Actions (`terraform fmt` + `terraform validate`).

## Lab: Secure Two-Tier VPC with Bastion and Private App Server

Located in [`aws-vpc-ec2/`](aws-vpc-ec2).

```text
                         Internet
                            │
                     Internet Gateway
                            │
   ┌────────────── VPC 10.0.0.0/16 ─────────────────┐
   │  Public 10.0.1.0/24          Public 10.0.2.0/24│
   │  ├─ Bastion (SSH from admin IP only)           │
   │  └─ NAT Gateway                                │
   │           │                                    │
   │  Private 10.0.11.0/24        Private 10.0.12.0/24
   │  └─ App server (nginx), SSH only from bastion  │
   └───────────── VPC Flow Logs → CloudWatch ───────┘
```

### What It Builds

| File | Resources |
|---|---|
| `vpc.tf` | VPC, 2 public + 2 private subnets across AZs, IGW, NAT Gateway, route tables, **VPC Flow Logs** |
| `security_groups.tf` | Bastion SG (SSH from one admin /32), app SG (SSH only from the bastion SG, HTTP only from the VPC) |
| `ec2.tf` | Bastion in a public subnet; nginx app server in a private subnet (IMDSv2 enforced) |
| `variables.tf` | Inputs with validation (rejects `0.0.0.0/0` for admin access) |
| `outputs.tf` | IDs and IPs for verification |

### Security Decisions
- The app server has **no public IP**. It is reachable only via the bastion and gets outbound access through NAT.
- Security groups reference other security groups instead of CIDRs.
- IMDSv2 is required on all instances, which blocks SSRF-style credential theft.
- Flow logs capture ACCEPT/REJECT traffic for troubleshooting and security investigations.
- State files and `*.tfvars` are git-ignored, so no secrets are committed.

### Usage

```bash
cd aws-vpc-ec2
cp terraform.tfvars.example terraform.tfvars   # set admin_cidr to your IP /32
terraform init
terraform plan
terraform apply

# Verify: from the bastion, reach the private app server
ssh -J ec2-user@$(terraform output -raw bastion_public_ip) ec2-user@$(terraform output -raw app_private_ip)
curl http://$(terraform output -raw app_private_ip)

terraform destroy   # NAT Gateway is billed hourly, so clean up
```

## Skills Demonstrated
- Terraform: providers, variables with validation, `count`, data sources, outputs
- AWS VPC design, routing, NAT, and security groups
- Security-by-default infrastructure (least privilege, IMDSv2, flow logs)
- CI for infrastructure code with GitHub Actions

## Related
- [aws-networking-labs](https://github.com/princewillsmith/aws-networking-labs): the same concepts built step by step with the AWS CLI
