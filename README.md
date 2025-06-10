# AWS CloudFormation Scalable Web Application Deployment with AWS CloudFormation

This repository contains CloudFormation templates and scripts to deploy a scalable web application infrastructure on AWS, including VPC, ALB, Auto Scaling Group, RDS database, S3 bucket, and monitoring resources.

For project video:
[Watch the video](https://drive.google.com/drive/folders/1oIzLf05Z9-1BtlrgQ_qVxGwGg4kLemkU)

For project Documentation:
[View the documentation](Documentation.md)

## Table of Contents

- [AWS Infrastructure](#aws-infrastructure)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Deployment Commands](#deployment-commands)
- [Accessing Resources](#accessing-resources)
- [Infrastructure Components](#infrastructure-components)
- [SSM Session Access](#ssm-session-access)
- [Load Balancer Access](#load-balancer-access)
- [Accessing the Database](#accessing-the-database)

## AWS Infrastructure

![AWS Infrastructure](Media/Scalable-Web-App-ALB-AS.png)
**AWS Infrastructure**

![AWS Infrastructure composer](Media/infrastructure-composer-web-app-CloudFormation.yml.png)
**AWS Infrastructure composer**

This project automates the deployment of a highly available web application using AWS CloudFormation. The infrastructure includes:

- **Multi-AZ VPC** with public and private subnets
- **Application Load Balancer** for traffic distribution
- **Auto Scaling Group** for EC2 instances
- **RDS MySQL database** in private subnets
- **S3 bucket** for storage
- **CloudWatch Alarms** and **SNS notifications**
- **IAM roles** for secure resource access

## Repository Structure

| File | Description |
|------|-------------|
| `create.sh` | Creates a CloudFormation stack |
| `update.sh` | Updates an existing CloudFormation stack |
| `web-app-CloudFormation.yml` | Main CloudFormation template |
| `web-app-CloudFormation-parameters.json` | Parameter values for the template |

## Prerequisites

1. **AWS CLI Configuration**:

   - Ensure you have configured the `saa-infra` profile with proper credentials

      ```bash
      aws configure list --profile saa-infra
      ```

2. **SSM Parameter Store**:
   - Database password must exist in SSM Parameter Store:

     ```bash
     aws ssm get-parameter --name "/DBPassword" --with-decryption
     ```

## Deployment Commands

1. **Validate Template**:

   ```bash
   aws cloudformation validate-template \
     --template-body file://web-app-CloudFormation.yml
   ```

2. **Create Stack**:

   ```bash
   ./create.sh manara-project-stack \
     web-app-CloudFormation.yml \
     web-app-CloudFormation-parameters.json
   ```

3. **Update Stack**:

   ```bash
   ./update.sh manara-project-stack \
     web-app-CloudFormation.yml \
     web-app-CloudFormation-parameters.json
   ```

   - `$1`: Stack name
   - `$2`: Template file
   - `$3`: Parameters file
   - Uses `CAPABILITY_IAM` for IAM resource creation
   - Deploys in eu-west-1 region

## Accessing Resources

1. **Web Application URL**:
   - After deployment, get the Load Balancer DNS name from stack outputs
   - Access via: `http://<ALB-DNS-Name>`

2. **EC2 Instance Access (SSM Session Manager)**:

   ```bash
   # List available instances
   aws ssm describe-instance-information \
     --query "InstanceInformationList[*].{InstanceId:InstanceId,Status:AgentStatus}" \
     --output table
   
   # Start session
   aws ssm start-session --target YOUR_INSTANCE_ID
   ```

## Infrastructure Components

| Resource | Description |
|----------|-------------|
| **VPC** | 10.0.0.0/16 with public/private subnets |
| **NAT Gateways** | Internet access for private subnets |
| **Application Load Balancer** | Distributes traffic to EC2 instances |
| **Auto Scaling Group** | Maintains 4-8 EC2 instances across AZs |
| **RDS MySQL** | Multi-AZ database in private subnets |
| **S3 Bucket** | Secure storage with versioning |
| **CloudWatch Alarms** | CPU monitoring for EC2 and RDS |
| **SNS Topic** | Email alerts to specified address |

### CloudWatch Alarms

- Monitor EC2 (10% CPU) and RDS (10% CPU)
- Trigger SNS notifications to registered email
- Adjust thresholds for production (80% recommended)

## Cleanup

Delete the CloudFormation stack via AWS Console or CLI to remove all resources.

### Notes

- Default CPU threshold is set to 10% for testing (adjust to 80% for production)
- RDS uses db.t3.micro (Free Tier eligible)
- All resources are tagged with `Manara-Project`
- Ensure you have the necessary IAM permissions to create and manage these resources

## SSM Session Access

```bash
aws ssm start-session --target YOUR_INSTANCE_ID
```

Provides secure SSH-less access to EC2 instances in private subnets.

## Load Balancer Access

```bash
http://<WebAppLBDNSName>
```

Access point for the web application (get DNS name from stack outputs).

### Important Security Notes

- Database password is securely retrieved from SSM Parameter Store
- RDS is not publicly accessible
- EC2 instances have minimal IAM permissions via IAM roles
- All resources are deleted when stack is removed

## Accessing the Database

The database (RDS MySQL instance) is deployed in **private subnets** for security and is **not publicly accessible**. Here's how to securely access it:

1. **Connect to a web server instance** using SSM Session Manager:

   ```bash
   aws ssm start-session --target YOUR_WEB_SERVER_INSTANCE_ID
   ```

2. **Install MySQL client** on the instance:

   ```bash
   sudo apt update
   sudo apt install mysql-client -y
   ```

3. **Connect to the RDS instance** using the MySQL client:

   ```bash
   $ mysql -h YOUR_RDS_ENDPOINT -u admin -p
     Enter password:
   ```

### Important Notes

1. **RDS Endpoint**: Find in CloudFormation stack outputs (key: `DBEndpoint`)
2. **Security Groups**:
   - Web servers can access RDS on port 3306
   - No public internet access to RDS
3. **Credentials**:
   - Username: Defined in parameters (`admin` by default)
   - Password: Stored securely in SSM Parameter Store (`/DBPassword`)
4. **IAM Role**: Ensure the web server instance has an IAM role with permissions to access SSM Parameter Store.

### Troubleshooting

- **Connection Issues**: Verify security group allows traffic from web server SG
- **Authentication Errors**: Confirm SSM parameter exists in same region
- **Timeout Issues**: Check NAT Gateway and route table configurations
