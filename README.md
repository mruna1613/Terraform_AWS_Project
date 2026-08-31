# AWS Load-Balanced Web App with Terraform

This project uses Terraform to provision a highly available web application on AWS. It creates a custom VPC with two public subnets across different Availability Zones, launches two EC2 instances running Apache, and distributes traffic between them using an Application Load Balancer (ALB).

## Architecture

```
                        Internet
                            │
                    Internet Gateway
                            │
                Application Load Balancer (ALB)
                            │
              ┌─────────────┴─────────────┐
              │                           │
        Subnet 1 (AZ-a)              Subnet 2 (AZ-b)
              │                           │
     EC2 Instance (web1)          EC2 Instance (web2)
      Apache + userdata.sh         Apache + userdata1.sh
```

**Components provisioned:**

- **VPC** — a custom VPC with a configurable CIDR block
- **Subnets** — two public subnets in different Availability Zones (`us-east-1a`, `us-east-1b`) with auto-assigned public IPs
- **Internet Gateway & Route Table** — routes `0.0.0.0/0` traffic through the IGW, associated with both subnets
- **Security Group** — allows inbound HTTP (80) and SSH (22), and unrestricted outbound traffic
- **EC2 Instances** — two `t2.micro` instances, one per subnet, each bootstrapped via a user data script that installs Apache and serves a simple HTML page showing the instance ID
- **Application Load Balancer** — internet-facing ALB spanning both subnets
- **Target Group & Attachments** — registers both instances behind the ALB with an HTTP health check on `/`
- **Listener** — forwards port 80 traffic from the ALB to the target group
- **S3 Bucket** — an example bucket resource included in the project

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/downloads) (v1.x recommended)
- An AWS account with credentials configured (e.g. via `aws configure` or environment variables)
- An existing EC2 key pair if you plan to SSH into the instances
- The AMI ID in `main.tf` (`ami-0b6d9d3d33ba97d99`) is region-specific — update it if deploying outside `us-east-1`

## Project Structure

```
.
├── main.tf         # Core infrastructure: VPC, subnets, ALB, EC2, security group, S3
├── variables.tf    # Input variables (e.g. cidr)
├── userdata.sh     # Bootstrap script for web1
├── userdata1.sh    # Bootstrap script for web2
└── README.md
```

> Note: `variables.tf` should define the `var.cidr` variable referenced in `main.tf` (e.g. `10.0.0.0/16`).

## Usage

1. Clone this repository:
   ```bash
   git clone <your-repo-url>
   cd <repo-folder>
   ```

2. Initialize Terraform:
   ```bash
   terraform init
   ```

3. Review the execution plan:
   ```bash
   terraform plan
   ```

4. Apply the configuration:
   ```bash
   terraform apply
   ```

5. Once applied, Terraform will output the ALB's DNS name:
   ```
   loadbalancerdns = "myalb-xxxxxxxxxx.us-east-1.elb.amazonaws.com"
   ```
   Open this URL in a browser and refresh a few times — you should see the page alternate between `web1` and `web2` as the ALB load-balances requests.

6. To tear everything down:
   ```bash
   terraform destroy
   ```

## What Each Instance Serves

Each EC2 instance runs a user data script (`userdata.sh` / `userdata1.sh`) that:
- Installs Apache and the AWS CLI
- Fetches the instance's own metadata (instance ID) via the EC2 metadata endpoint
- Generates a simple HTML page displaying the instance ID, so you can visually confirm the ALB is distributing traffic across both instances
- Starts and enables Apache on boot

## Security Notes

- The security group currently allows SSH (22) and HTTP (80) from `0.0.0.0/0`. For any real-world use, restrict SSH access to your own IP range.
- No HTTPS/TLS is configured on the ALB listener — this project focuses on demonstrating load balancing across AZs, not production-hardened networking.
- The S3 bucket name is hardcoded and globally unique bucket names in AWS mean you'll need to change it before applying if you reuse this configuration.

## Possible Improvements

- Add an Auto Scaling Group instead of static instance count
- Add HTTPS listener with an ACM certificate
- Parameterize AMI ID, instance type, and AZs as variables
- Restrict SSH ingress to a specific CIDR
- Add a `terraform.tfvars.example` file for easy variable configuration

## Author

Built by Mrunmayee as part of a hands-on Terraform/AWS learning project.
