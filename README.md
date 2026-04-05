# Ubuntu Test Environment on AWS — CloudFormation

> **"I just need a box to test this on."**

Sometimes you don't have a full CI/CD pipeline with Terraform, remote state, and a module registry ready to go. You just need to quickly spin up a clean Ubuntu server in AWS to experiment with **Docker**, **Ansible**, or any other tool — and tear it down when you're done. This CloudFormation template is exactly that: a single YAML file you run once, get an SSH-ready Ubuntu instance, and move on.

No Terraform. No state files. No pipeline. Just a working server in under five minutes.

---

## What it provisions

```
AWS Account
└── VPC (10.0.0.0/16)
    └── Public Subnet (10.0.1.0/24)
        ├── Internet Gateway + Route Table
        ├── Security Group (SSH inbound, all outbound)
        └── EC2 Ubuntu 22.04 LTS (t3.micro by default)
            ├── Encrypted gp3 EBS root volume (20 GB)
            ├── IMDSv2 enforced
            ├── IAM role with SSM Session Manager
            └── SSH hardened via UserData
```

---

## Prerequisites

- AWS CLI installed and configured (`aws configure`)
- An existing **EC2 Key Pair** in the target region
- IAM permissions to create VPCs, EC2 instances, IAM roles, and CloudFormation stacks

---

## Quick start

**1. Clone or download the template**

```bash
git clone <your-repo-url>
cd <your-repo>
```

**2. Find your public IP** (to lock down SSH access)

```bash
curl -s https://checkip.amazonaws.com
# → 203.0.113.10
```

**3. Deploy the stack**

```bash
aws cloudformation deploy \
  --template-file ubuntu-vpc.yaml \
  --stack-name ubuntu-test-env \
  --parameter-overrides \
      KeyPairName=my-key \
      AllowedSSHCIDR=203.0.113.10/32 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-west-1
```

**4. Get the instance IP and connect**

```bash
aws cloudformation describe-stacks \
  --stack-name ubuntu-test-env \
  --query "Stacks[0].Outputs" \
  --output table

ssh -i ~/.ssh/my-key.pem ubuntu@<PublicIP>
```

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `EnvironmentName` | `dev` | Prefix used in all resource names and tags |
| `VpcCIDR` | `10.0.0.0/16` | CIDR block for the VPC |
| `PublicSubnetCIDR` | `10.0.1.0/24` | CIDR block for the public subnet |
| `AllowedSSHCIDR` | `0.0.0.0/0` | **Replace with your IP** — CIDR allowed to reach port 22 |
| `InstanceType` | `t3.micro` | EC2 instance type (`t3.micro`, `t3.small`, `t3.medium`) |
| `KeyPairName` | *(required)* | Name of your existing EC2 key pair |
| `UbuntuAMI` | SSM-resolved | Latest Ubuntu 22.04 LTS AMI, auto-resolved per region |

---

## Security features

| Feature | What it does |
|---|---|
| **`AllowedSSHCIDR`** | Restricts SSH to a specific IP range. Default is open (`0.0.0.0/0`) — lock it down to your IP before deploying |
| **IMDSv2 enforced** | `HttpTokens: required` blocks SSRF-based credential theft via the metadata endpoint |
| **EBS encryption** | Root volume is AES-256 encrypted at rest |
| **SSH hardening** | UserData disables root login, password auth, and X11 forwarding on first boot |
| **SSM Session Manager** | IAM role attached to the instance allows browser/CLI-based shell access without opening any ports at all |

---

## Example use cases

### Test Docker

```bash
ssh -i ~/.ssh/my-key.pem ubuntu@<PublicIP>

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker

# Run something
docker run --rm hello-world
docker run -d -p 80:80 nginx
```

### Run an Ansible playbook against the instance

```ini
# inventory.ini
[test]
<PublicIP> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/my-key.pem
```

```bash
ansible -i inventory.ini test -m ping
ansible-playbook -i inventory.ini my-playbook.yml
```

### Use SSM Session Manager (no SSH key needed)

```bash
aws ssm start-session --target <InstanceId> --region eu-west-1
```

---

## Tear it down

When you're done experimenting, delete the entire stack — no manual cleanup needed:

```bash
aws cloudformation delete-stack \
  --stack-name ubuntu-test-env \
  --region eu-west-1
```

All resources created by the template (VPC, subnets, security groups, EC2 instance, IAM role) are deleted automatically.

---

## Useful CLI snippets

```bash
# Watch stack events in real time
aws cloudformation describe-stack-events \
  --stack-name ubuntu-test-env \
  --query "StackEvents[*].[Timestamp,ResourceStatus,LogicalResourceId,ResourceStatusReason]" \
  --output table

# Get just the public IP
aws cloudformation describe-stacks \
  --stack-name ubuntu-test-env \
  --query "Stacks[0].Outputs[?OutputKey=='PublicIP'].OutputValue" \
  --output text

# Update the stack (e.g. change instance type)
aws cloudformation deploy \
  --template-file ubuntu-vpc.yaml \
  --stack-name ubuntu-test-env \
  --parameter-overrides InstanceType=t3.small \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## Cost

Running a `t3.micro` in `eu-west-1` costs roughly **$0.0134/hour** (~$0.32/day). The 20 GB gp3 volume adds ~$0.05/day. Always delete the stack when you're finished.

---

## License

MIT — use freely, modify as needed.
