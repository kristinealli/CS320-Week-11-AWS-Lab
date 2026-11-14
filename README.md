# CS320 – Week 11: AWS Network Automation with Ansible

## Project Overview

This project uses Ansible to automate the creation and management of basic network infrastructure in Amazon Web Services (AWS). The deployment creates a Virtual Private Cloud (VPC), public and private subnets, an internet gateway, route tables, and a security group for a future web server.

The project demonstrates modular programming through reusable Ansible roles, centralized variables, automated infrastructure provisioning, and idempotence testing.

## Learning Objectives

- Configure Ansible to authenticate with AWS.
- Provision AWS networking resources using Ansible.
- Organize automation into reusable roles.
- Configure public and private subnets.
- Manage network routing and security groups.
- Test deployment idempotence.
- Remove cloud resources through automation.

## Network Architecture

| Component | Configuration |
|---|---|
| AWS Region | us-east-2 (Ohio) |
| VPC | 10.0.0.0/16 |
| Public Subnet | 10.0.1.0/24 |
| Private Subnet | 10.0.2.0/24 |
| Public Availability Zone | us-east-2a |
| Private Availability Zone | us-east-2b |
| Internet Gateway | Attached to the VPC |
| Public Route Table | Local route and 0.0.0.0/0 via internet gateway |
| Private Route Table | Local VPC routing only |
| Security Group | week11-web-sg |

### Public and Private Subnets

The public subnet is configured to automatically assign public IPv4 addresses to eligible resources and uses a route table containing a default route (`0.0.0.0/0`) through the internet gateway. This allows appropriately configured future resources to communicate with the internet.

The private subnet uses a separate route table without a default internet route. It supports internal VPC communication but does not provide direct internet access.

## Project Structure

```text
Week 11 AWS Lab/
├── .venv/                    # Local, Git-ignored
├── ansible.cfg
├── requirements.yml
├── group_vars/
│   └── all.yml
├── local_vars.yml            # Local, Git-ignored
├── local_vars.yml.example
├── roles/
│   ├── vpc/
│   │   └── tasks/main.yml
│   ├── subnets/
│   │   └── tasks/main.yml
│   ├── gateway/
│   │   └── tasks/main.yml
│   └── security_group/
│       └── tasks/main.yml
├── site.yml
├── test_idempotence.yml
├── destroy.yml
├── evidence/
├── .gitignore
└── README.md
```

The project uses separate roles for deployment and standalone playbooks for testing and cleanup.

The virtual environment and `local_vars.yml` are excluded from Git. The example variables file can be committed without including account-specific values.

## Requirements and Dependencies

The project requires:

- Python 3 and a virtual environment
- Ansible
- AWS CLI v2 supporting `aws login`
- `amazon.aws` Ansible collection
- `boto3`, `botocore`, and `awscrt`
- AWS account with appropriate permissions
- Git

Dependencies used during development:

| Dependency | Version |
|---|---|
| boto3 | 1.44.3 |
| botocore | 1.44.3 |
| amazon.aws | 11.5.1 |

### Environment Setup

Create and activate the virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install the Python dependencies:

```bash
python -m pip install ansible boto3 "botocore[crt]"
```

Install the AWS Ansible collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

The AWS CLI must be installed separately.

## AWS Authentication and Credential Security

AWS authentication was configured using browser-based AWS CLI login with a named profile.

```bash
aws login --profile aws-lab
```

Authentication was verified using:

```bash
aws sts get-caller-identity --profile aws-lab
```

Before running Ansible, select the profile and region:

```bash
export AWS_PROFILE=aws-lab
export AWS_REGION=us-east-2
```

### Credential Protection

Temporary AWS credentials are managed through AWS CLI authentication.

Security precautions include:

- Keeping AWS access keys, secret keys, and session tokens out of YAML files.
- Using a named AWS CLI profile for authentication.
- Excluding the Python virtual environment and local configuration from Git.
- Keeping the public SSH source IP and expected AWS account ID in a Git-ignored local variables file.
- Restricting SSH access to a single public IPv4 address.
- Using explicit confirmation and resource identity checks before deletion.
- Avoiding unnecessary AWS services and resources to limit costs.

Ansible Vault was not required because the project does not store AWS authentication secrets.

## Ansible Roles

### 1. VPC Role

**Location:** `roles/vpc/tasks/main.yml`

Uses `amazon.aws.ec2_vpc_net` to create the VPC, enable DNS support, assign resource tags, and register the VPC ID.

### 2. Subnets Role

**Location:** `roles/subnets/tasks/main.yml`

Uses `amazon.aws.ec2_vpc_subnet` to create two subnets:

- Public subnet in `us-east-2a`, with automatic public IPv4 assignment enabled.
- Private subnet in `us-east-2b`, without automatic public IPv4 assignment.

The role registers both subnet IDs for later tasks.

### 3. Gateway Role

**Location:** `roles/gateway/tasks/main.yml`

Uses `amazon.aws.ec2_vpc_igw` and `amazon.aws.ec2_vpc_route_table`.

The role creates and attaches an internet gateway, configures a public route table with an internet gateway route, and configures a private route table with local VPC routing only.

Each subnet is associated with its corresponding route table.

### 4. Security Group Role

**Location:** `roles/security_group/tasks/main.yml`

Uses `amazon.aws.ec2_security_group` to create a security group intended for a future web server.

| Protocol | Port | Allowed Source |
|---|---|---|
| TCP | 80 (HTTP) | 0.0.0.0/0 |
| TCP | 443 (HTTPS) | 0.0.0.0/0 |
| TCP | 22 (SSH) | Restricted public IPv4 /32 |

Outbound traffic is permitted.

## Project Variables

The project separates reusable configuration from values specific to the deployment environment.

### Shared Variables

The `group_vars/all.yml` file contains nonsecret configuration that can be committed to Git:

```yaml
aws_region: us-east-2

vpc_name: week11-ansible-vpc
vpc_cidr: 10.0.0.0/16

public_subnet_name: week11-public
public_subnet_cidr: 10.0.1.0/24

private_subnet_name: week11-private
private_subnet_cidr: 10.0.2.0/24

internet_gateway_name: week11-igw
public_route_table_name: week11-public-rt
private_route_table_name: week11-private-rt

security_group_name: week11-web-sg
```

### Local Variables

The `local_vars.yml` file contains environment-specific values and is excluded from Git.

The repository includes `local_vars.yml.example` as a template:

```yaml
---
ssh_allowed_cidr: "YOUR_PUBLIC_IP/32"
expected_aws_account_id: "YOUR_12_DIGIT_ACCOUNT_ID"
```

To obtain the public IPv4 address:

```bash
curl -4 https://checkip.amazonaws.com
```

Append `/32` to restrict SSH access to that address.

To obtain the AWS account ID:

```bash
aws sts get-caller-identity \
    --profile aws-lab \
    --query Account \
    --output text
```

These identifiers are not authentication secrets, but are kept in a local file to avoid unnecessarily publishing environment-specific information. Update example and rename it to `local_vars.yml` before deployment.

## Deployment Instructions

Activate the virtual environment and select the AWS profile:

```bash
source .venv/bin/activate
export AWS_PROFILE=aws-lab
export AWS_REGION=us-east-2
```

Create `local_vars.yml` from the example and populate the two required values.

Check playbook syntax:

```bash
ansible-playbook -i 'localhost,' site.yml --syntax-check
```

Deploy the infrastructure:

```bash
ansible-playbook -i 'localhost,' site.yml \
    -e @local_vars.yml
```

The main playbook executes the roles in dependency order:

1. VPC
2. Subnets
3. Internet gateway and route tables
4. Security group

This order ensures that resources are created after their dependencies exist.

## Idempotence Testing

Idempotence means that running the same Ansible playbook multiple times produces the same desired configuration without unnecessary changes or duplicate resources.

After the initial deployment, the playbook was executed again without modifying the infrastructure.

The second run reported:

```text
PLAY RECAP
localhost : ok=16 changed=0 unreachable=0 failed=0
```

This confirms that Ansible recognized the existing resources and did not need to modify them.

The project also includes `test_idempotence.yml` for automated testing and evidence collection. Its nested deployment commands must receive the same local variables used during deployment.

The resulting logs are saved in the `evidence/` directory.

## Resource Cleanup

All resources created for the assignment must be removed after verification and evidence collection.

The `destroy.yml` playbook is designed to:

1. Require explicit authorization.
2. Verify the AWS account and region.
3. Discover the lab resources using their names and project tags.
4. Validate that the resources belong to the intended VPC.
5. Remove resources in dependency order.
6. Confirm that the lab VPC no longer exists.

The cleanup order is:

1. Security group
2. Public and private subnets
3. Custom route tables
4. Internet gateway
5. VPC

The playbook uses an explicit `confirm_destroy` variable to prevent accidental execution.

After saving all deployment evidence, run:

```bash
mkdir -p evidence
set -o pipefail

ansible-playbook -i 'localhost,' destroy.yml \
    -e @local_vars.yml \
    -e confirm_destroy=true \
    2>&1 | tee evidence/destroy.log
```

The `tee` command displays the cleanup output and saves it to `evidence/destroy.log`.

After cleanup, verify that the VPC and its related resources no longer exist in AWS.

