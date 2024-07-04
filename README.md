# Automated Deployment of k3s on AWS Using GitHub Actions, Pulumi, and Ansible

## Overview

This project aims to automate the deployment of a lightweight Kubernetes distribution (k3s) on AWS using `GitHub Actions` for orchestration, `Pulumi` for infrastructure as code (IaC), and `Ansible` for configuration management.

![alt text](./images/image-2.png)

### Key Components

- **GitHub Actions**: Automates workflows triggered by events in GitHub repositories.
- **Pulumi**: Infrastructure as Code tool to provision AWS resources.
- **Ansible**: Configuration management tool for installing and configuring k3s on AWS EC2 instances.

### Prerequisites

Before proceeding, ensure you have the following:

- An AWS account with appropriate IAM permissions.
- Pulumi CLI installed locally and authenticated with your Pulumi account and a PULUMI access token.
- GitHub repository set up to host your Pulumi project.
- Access to GitHub Secrets to securely store credentials and sensitive information.

## Project Structure
The project is organized into the following structure:

```sh
K3s-deployment-automation/
├── Infra/                    # Pulumi project directory
│   ├── __main__.py           # Pulumi Python script defining AWS resources
│   ├── Pulumi.dev.yaml
│   ├── Pulumi.yaml
│   ├── requirements.txt
│   ├── venv/                 # Virtual environment for Python dependencies 
│   └── ...                   # Other Pulumi project files and configurations
├── .github/                  # GitHub Actions workflows directory
│   └── workflows/
│       ├── infra.yml         # GitHub Actions workflow for deploying infrastructure
│       └── setup-git-runner.yml    # GitHub Actions workflow for deploying K3s
├── ...                       # Other project files and directories
```

## Deployment Process
The deployment process involves the following steps:
1. **GitHub Actions 1**: A GitHub Actions workflow is triggered by a push event to the main branch
2. **Pulumi**: The Pulumi Python script is executed to provision the AWS resources
4. **GitHub Action 2**: A second GitHub Actions workflow is triggered to install self-hosted Git runner and install Ansible in the Git-runner public instance making it the control node for Ansible automation
5. **GitHUb Action 3**: The Ansible playbook is executed to install and configure k3s on the private EC2 instances.

![alt text](./images/image-1.png)


## Setup Pulumi Project

1. **Initialize Pulumi Project**

   - Create a new directory for your Pulumi project:

     ```bash
     mkdir Infra
     cd Infra
     ```

   - Initialize a new Pulumi AWS Python project:

     ```bash
     pulumi new aws-python
     ```

2. **Define AWS Resources**

Write Python scripts (`__main__.py` or similar) to define AWS resources using Pulumi's AWS SDK.

```python
import os
import pulumi
import pulumi_aws as aws
import pulumi_aws.ec2 as ec2
from pulumi_aws.ec2 import SecurityGroupRuleArgs


# Configuration
config = pulumi.Config()
instance_type = 't3.small'
ami = "ami-003c463c8207b4dfa"


# Create a VPC
vpc = ec2.Vpc(
    'my-vpc',
    cidr_block='10.0.0.0/16',
    enable_dns_hostnames=True,
    enable_dns_support=True
)

# Create subnets
public_subnet = ec2.Subnet('public-subnet',
    vpc_id=vpc.id,
    cidr_block='10.0.1.0/24',
    map_public_ip_on_launch=True,
    availability_zone='ap-southeast-1a'
)

private_subnet = ec2.Subnet('private-subnet',
    vpc_id=vpc.id,
    cidr_block='10.0.2.0/24',
    map_public_ip_on_launch=False,
    availability_zone='ap-southeast-1a'
)

# Internet Gateway
igw = ec2.InternetGateway('internet-gateway', vpc_id=vpc.id)

# Route Table for Public Subnet
public_route_table = ec2.RouteTable('public-route-table', 
    vpc_id=vpc.id,
    routes=[{
        'cidr_block': '0.0.0.0/0',
        'gateway_id': igw.id,
    }]
)

# Associate the public route table with the public subnet

public_route_table_association = ec2.RouteTableAssociation(
    'public-route-table-association',
    subnet_id=public_subnet.id,
    route_table_id=public_route_table.id
)

# Elastic IP for NAT Gateway
eip = ec2.Eip('nat-eip', vpc=True)

# NAT Gateway
nat_gateway = ec2.NatGateway(
    'nat-gateway',
    subnet_id=public_subnet.id,
    allocation_id=eip.id
)

# Route Table for Private Subnet
private_route_table = ec2.RouteTable(
    'private-route-table', 
    vpc_id=vpc.id,
    routes=[{
        'cidr_block': '0.0.0.0/0',
        'nat_gateway_id': nat_gateway.id,
    }]
)

# Associate the private route table with the private subnet
private_route_table_association = ec2.RouteTableAssociation(
    'private-route-table-association',
    subnet_id=private_subnet.id,
    route_table_id=private_route_table.id
)

# Security Group for allowing SSH and k3s traffic
security_group = aws.ec2.SecurityGroup("web-secgrp",
    description='Enable SSH and K3s access',
    vpc_id=vpc.id,
    ingress=[
        {
            "protocol": "tcp",
            "from_port": 22,
            "to_port": 22,
            "cidr_blocks": ["0.0.0.0/0"],
        },
        {
            "protocol": "tcp",
            "from_port": 6443,
            "to_port": 6443,
            "cidr_blocks": ["0.0.0.0/0"],
        },
    ],
    egress=[{
        "protocol": "-1",
        "from_port": 0,
        "to_port": 0,
        "cidr_blocks": ["0.0.0.0/0"],
    }],
)

#key pair
public_key = os.getenv("PUBLIC_KEY")

# Create the EC2 KeyPair using the public key
key_pair = aws.ec2.KeyPair("my-key-pair",
    key_name="my-key-pair",
    public_key=public_key)

# EC2 instances
master_instance = ec2.Instance(
    'master-instance',
    instance_type=instance_type,
    ami=ami,
    subnet_id=private_subnet.id,
    vpc_security_group_ids=[security_group.id],
    key_name=key_pair.key_name,
    tags={
        'Name': 'Master Node',
    }
)

worker_instance_1 = ec2.Instance('worker-instance-1',
    instance_type=instance_type,
    ami=ami,
    subnet_id=private_subnet.id,
    vpc_security_group_ids=[security_group.id],
    key_name=key_pair.key_name,
    tags={
        'Name': 'Worker Node 1',
    }
)

worker_instance_2 = ec2.Instance('worker-instance-2',
    instance_type=instance_type,
    ami=ami,
    subnet_id=private_subnet.id,
    vpc_security_group_ids=[security_group.id],
    key_name=key_pair.key_name,
    tags={
        'Name': 'Worker Node 2',
    }
)

git_runner_instance = ec2.Instance('git-runner-instance',
    instance_type=instance_type,
    ami=ami,
    subnet_id=public_subnet.id,
    vpc_security_group_ids=[security_group.id],
    key_name=key_pair.key_name,
    tags={
        'Name': 'Git Runner',
    }
)

# Output the instance public IP addresses
pulumi.export('git_runner_public_ip', git_runner_instance.public_ip)
pulumi.export('master_private_ip', master_instance.private_ip)
pulumi.export('worker1_private_ip', worker_instance_1.private_ip)
pulumi.export('worker2_private_ip', worker_instance_2.private_ip)
```

3. **Commit Pulumi Project to GitHub**

Commit your Pulumi project to your GitHub repository for version control and automated deployment.

## Configure secrets

### 1. Generate SSH Keys Locally and save as a github secrets

Generate a new SSH key pair on your local machine. This key pair will be used to SSH into the EC2 instances.

```sh
ssh-keygen -t ed25519 -C "default"
```

This will generate two files, typically in the `~/.ssh` directory:
- `id_ed25519` (private key)
- `id_ed25519.pub` (public key)

### 2. Go to the SSH Folder

Navigate to the `.ssh` directory where the keys were generated.

```sh
cd ~/.ssh
```

### 3. Get the Public Key and Add It to GitHub Secrets

1. Open the `id_ed25519.pub` file and copy its contents.
   
   ```sh
   cat id_ed25519.pub
   ```
2. Open the `id_ed25519` file and copy its contents.
   
   ```sh
   cat id_ed25519
   ```
   ![alt text](./images/image.png)

## Save secrets and AWS credentials as Github secrets

1. Go to your GitHub repository.
2. Navigate to **Settings** > **Secrets and variables** > **Actions** > **New repository secret**.
3. Add these secrets named:

  - `PUBLIC_KEY` -> `id_ed25519.pub`

  - `SSH_PRIVATE_KEY` -> `id_ed25519`

  - `AWS_ACCESS_KEY_ID` -> `AWS access key`

  - `AWS_SECRET_ACCESS_KEY` -> `AWS secret key`

  - `PULUMI_ACCESS_TOKEN` -> `Pulumi access key`

  ![Github secret](https://github.com/Konami33/k3s-deployment-automation/blob/main/images/secrets.png?raw=true)

## Configure GitHub Actions for Infrastructure Deployment

1. **Create GitHub Actions Workflow (`infra.yml`)**

This workflow will create the required AWS Infrastructure. It will create VPC, public-subnet, private-subnet, route-table with subent association, internet gateway, NAT gateway, security group, and Instances. 

```yaml
name: Deploy Infrastructure

on:
  push:
    branches:
      - main
    paths:
      - Infra/**

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.x'

      - name: Install dependencies
        run: |
          python -m venv venv
          source venv/bin/activate
          pip install pulumi pulumi-aws

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Pulumi login
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
        run: pulumi login

      - name: Pulumi stack select
        run: pulumi stack select Konami33/k3s_infra/dev --cwd Infra

      - name: Set public key environment variable
        run: echo "PUBLIC_KEY=${{ secrets.PUBLIC_KEY }}" >> $GITHUB_ENV

      - name: Pulumi refresh
        run: pulumi refresh --yes --cwd Infra

      - name: Pulumi up
        run: pulumi up --yes --cwd Infra

      - name: Save Pulumi outputs
        id: pulumi_outputs
        run: |
          GIT_RUNNER_IP=$(pulumi stack output git_runner_public_ip --cwd Infra)
          MASTER_IP=$(pulumi stack output master_private_ip --cwd Infra)
          WORKER1_IP=$(pulumi stack output worker1_private_ip --cwd Infra)
          WORKER2_IP=$(pulumi stack output worker2_private_ip --cwd Infra)
          
          echo "GIT_RUNNER_IP=$GIT_RUNNER_IP" >> $GITHUB_ENV
          echo "MASTER_IP=$MASTER_IP" >> $GITHUB_ENV
          echo "WORKER1_IP=$WORKER1_IP" >> $GITHUB_ENV
          echo "WORKER2_IP=$WORKER2_IP" >> $GITHUB_ENV
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
```

2. Create the GitHub action workflow `setup-git-runner.yml`:

This GitHub Actions workflow will run after the `Deploy Infrastructure` workflow completes. Its primary purpose is to configure a GitHub Runner on an EC2 instance, set up Ansible on that instance, and prepare the environment for deploying a k3s cluster using Ansible.

1. **Triggering the Workflow:**
    - The workflow is triggered when the `Deploy Infrastructure` workflow completes.

2. **Setting Up the Environment:**
    - The workflow runs on a GitHub-hosted runner using the latest Ubuntu environment.

3. **Checking Out the Code:**
    - The repository code is checked out to the runner.

4. **Configuring AWS Credentials:**
    - AWS credentials are configured using secrets stored in the GitHub repository to allow Pulumi to interact with AWS.

5. **Pulumi Operations:**
    - Pulumi is logged into using an access token.
    - The public IP of the GitHub Runner EC2 instance and the private IPs of the master and worker nodes are retrieved and stored as environment variables.

6. **Setting Up SSH:**
    - An SSH agent is set up, and the private key stored in GitHub secrets is added to it for secure connections.

7. **Configuring the GitHub Runner and Installing Ansible:**
    - The workflow SSHs into the GitHub Runner EC2 instance.
    - Installs Ansible on the instance.
    - Sets up the GitHub Runner
    - Starts the GitHub Runner service.

8. **Copying Ansible Directory:**
    - The `ansible` directory from the GitHub Actions runner is copied to the GitHub Runner EC2 instance.

```yml
name: Setup GitHub Runner and Ansible

on:
  workflow_run:
    workflows: ["Deploy Infrastructure"]
    types:
      - completed

jobs:
  setup_runner:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Pulumi login
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
        run: pulumi login

      - name: Pulumi stack select
        run: pulumi stack select Konami33/k3s_infra/dev --cwd Infra

      - name: Pulumi refresh
        run: pulumi refresh --yes --cwd Infra
      
      - name: Save Pulumi outputs
        id: pulumi_outputs
        run: |
          GIT_RUNNER_IP=$(pulumi stack output git_runner_public_ip --cwd Infra)
          MASTER_NODE_IP=$(pulumi stack output master_private_ip --cwd Infra)
          WORKER_NODE1_IP=$(pulumi stack output worker1_private_ip --cwd Infra)
          WORKER_NODE2_IP=$(pulumi stack output worker2_private_ip --cwd Infra)
          echo "GIT_RUNNER_IP=$GIT_RUNNER_IP" >> $GITHUB_ENV
          echo "MASTER_NODE_IP=$MASTER_NODE_IP" >> $GITHUB_ENV
          echo "WORKER_NODE1_IP=$WORKER_NODE1_IP" >> $GITHUB_ENV
          echo "WORKER_NODE2_IP=$WORKER_NODE2_IP" >> $GITHUB_ENV
      
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}

      - name: Set up SSH agent
        uses: webfactory/ssh-agent@v0.5.3
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: SSH to Runner EC2 and install GitHub Runner and Ansible
        run: |
          ssh -o StrictHostKeyChecking=no ubuntu@${{ env.GIT_RUNNER_IP }} << EOF
          sudo apt-get update -y
          sudo apt install software-properties-common
          sudo apt-add-repository --yes --update ppa:ansible/ansible
          sudo apt-get install -y ansible
          ansible --version

          mkdir actions-runner && cd actions-runner

          curl -o actions-runner-linux-x64-2.317.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz

          echo "9e883d210df8c6028aff475475a457d380353f9d01877d51cc01a17b2a91161d  actions-runner-linux-x64-2.317.0.tar.gz" | shasum -a 256 -c

          tar xzf ./actions-runner-linux-x64-2.317.0.tar.gz

          ./config.sh --url https://github.com/Konami33/k3s-deployment --token AQGGSGXEHNCEG7HR5YI4D4LGQXPVG --name "Git Runner 2"

          sudo ./svc.sh install
          sudo ./svc.sh start
          EOF

      - name: Copy Ansible directory to GitHub Runner
        run: |
          scp -o StrictHostKeyChecking=no -r $GITHUB_WORKSPACE/ansible ubuntu@${{ env.GIT_RUNNER_IP }}:/home/ubuntu/

      - name: SSH into GitHub Runner and run Ansible playbook
        run: |
          ssh -o StrictHostKeyChecking=no ubuntu@${{ env.GIT_RUNNER_IP }} << 'EOF'

          # Create inventory file with dynamic IPs
          cat <<EOT > /home/ubuntu/ansible/inventory/hosts.ini
          [master]
          master-node ansible_host=${{ env.MASTER_NODE_IP }}

          [workers]
          worker-node-1 ansible_host=${{ env.WORKER_NODE1_IP }}
          worker-node-2 ansible_host=${{ env.WORKER_NODE2_IP }}

          [k3s_cluster:children]
          master
          workers
          EOT  

```

## SSH intro Git-Runner public instance

1. Open a Ubuntu terminal and change directory to your private key directory. Generally this directory is `~/.ssh`

```sh
cd ~/.ssh
```
2. Now Run this command to generate a pem file

```sh
cp ./id_rsa ./my-key-pair.pem
```
3. Now using this pem file SSH into the Git-runner Instance

```sh
ssh -i my-key-pair.pem ubuntu@<git-runner-instance-public-ip>
```

![alt text](image.png)

## Connect all private instances (Master Node, Worker Nodes) using a new key

1. Create a new rsa key

```sh
ssh-keygen -t rsa -b 2048
```

2. Run this command for ssh copying the Public key into remote instances

```sh
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@master-node-ip
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@worker-node-1-ip
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@master-node-2-ip
```
3. Now you can ssh into any instances using this command:

```sh
ssh ubuntu@ubuntu@master-node-ip
ssh ubuntu@worker-node-1-ip
ssh ubuntu@worker-node-2-ip
```

## Run the ansible playbook for K3s installation

1. Goto ansible directory and run:

```sh
cd ansible
ansible-playbook -i inventory/hosts.ini site.yml
```
This will install k3s in the master node ec2 and connect the worker node ec2 as a worker node.

```sh
ubuntu@ip-10-0-1-221:~/ansible$ ansible-playbook -i inventory/hosts.ini site.yml

PLAY [k3s_cluster] *******************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************
ok: [master-node]
ok: [worker-node-2]
ok: [worker-node-1]

TASK [common : Update and upgrade apt packages] **************************************************************************************************
changed: [master-node]
changed: [worker-node-1]
changed: [worker-node-2]

TASK [common : Install dependencies] *************************************************************************************************************
changed: [worker-node-2]
changed: [master-node]
changed: [worker-node-1]

PLAY [master] ************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************
ok: [master-node]

TASK [k3s-master : Download and install k3s] *****************************************************************************************************
changed: [master-node]

TASK [k3s-master : Set file permission] **********************************************************************************************************
changed: [master-node]

TASK [k3s-master : Get k3s token] ****************************************************************************************************************
ok: [master-node]

TASK [k3s-master : Get master node IP] ***********************************************************************************************************
ok: [master-node]

PLAY [workers] ***********************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************
ok: [worker-node-2]
ok: [worker-node-1]

TASK [k3s-worker : Join the k3s cluster] *********************************************************************************************************
changed: [worker-node-1]
changed: [worker-node-2]

PLAY RECAP ***************************************************************************************************************************************
master-node                : ok=8    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
worker-node-1              : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
worker-node-2              : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

ubuntu@ip-10-0-1-221:~/ansible$
```
## varification
You can also check if k3s is installed successfull or not.
1. SSH into Masternode
```sh
ssh ubuntu@master-node-ip
```
2. Now you are in the master node instance. Run this command
```sh
kubectl get nodes
```
![alt text](<Screenshot 2024-07-04 054420.png>)

here we can see, one master node and two worker node.

So we have deployed k3s using ansible automation from the Control node(Git-runner)