# Kubernetes the Hard Way: Initial Setup

## Setting Up Amazon Web Services (AWS) Command Line Interface (CLI)

![](./images/aws.drawio.svg)

### Step 1: Configure AWS CLI

Before interacting with AWS services, you must configure the AWS CLI with your credentials. This can be done using the `aws configure` command, which prompts you for four pieces of information:

1. **AWS Access Key ID**: Your access key to authenticate AWS API requests.
2. **AWS Secret Access Key**: A secret key associated with your access key.
3. **Default region**: The AWS region in which you want to provision your resources (`ap-southeast-1`).
4. **Default output format**: You can choose the output format (JSON, text, table).

Run the following command and provide the required information:

```sh
aws configure
```

After configuration, the AWS CLI will use these credentials to manage resources in your chosen region.

### Step 2: Install `jq` for JSON Parsing

`jq` is a lightweight and flexible command-line JSON processor that is helpful for parsing JSON responses from AWS commands.

#### Installation

Refer to the official jq [download page](https://jqlang.github.io/jq/download/) and follow the instructions for your operating system. On a Debian-based system like Ubuntu, you can use:

```sh
sudo apt-get update
sudo apt-get install jq -y
```

This will install `jq` on your system, allowing you to parse AWS CLI responses more easily.

## Installing Client Tools

For the Kubernetes the Hard Way tutorial, you'll need the following command-line utilities:

1. **cfssl**: A command-line tool used to build and manage a Public Key Infrastructure (PKI).
2. **cfssljson**: A tool for working with JSON output from cfssl.
3. **kubectl**: The command-line tool for interacting with a Kubernetes cluster.

### Step 1: Install CFSSL and CFSSLJSON

CFSSL (Cloudflare’s PKI and TLS toolkit) will be used for managing the PKI and generating TLS certificates required by Kubernetes components.

#### Download and Install

Use the following commands to download and install `cfssl` and `cfssljson`:

```sh
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64

sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

#### Verification

Ensure the tools have been installed successfully by checking their versions:

```sh
cfssl version
```

You should see output similar to:

```sh
Version: 1.4.1
Runtime: go1.12.12
```

Similarly, check for `cfssljson`:

```sh
cfssljson --version
```

### Step 2: Install Kubectl

`kubectl` is the primary tool for interacting with the Kubernetes cluster. You'll use it to deploy applications, inspect cluster resources, and manage Kubernetes objects.

#### Download and Install

Use the following commands to install `kubectl`:

```sh
curl -LO "https://dl.k8s.io/release/v1.21.0/bin/linux/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```

#### Verification

Check that `kubectl` is installed correctly by verifying its version:

```sh
kubectl version --client
```

## Provisioning Compute Resources

In this step, you will provision the necessary AWS resources that will host your Kubernetes cluster.

### Step 1: Create a Directory for Your Infrastructure

Before starting, it’s best to create a dedicated directory for the infrastructure files:

```sh
mkdir k8s-infra-aws
cd k8s-infra-aws
```

### Step 2: Install Python `venv`

To manage Python environments easily, you will need to install the `venv` module. Run the following commands to install it:

```sh
sudo apt update
sudo apt install python3.8-venv -y
```

This will set up a Python virtual environment which will be useful later when working with Pulumi.

### Step 3: Initialize a New Pulumi Project

Pulumi is an Infrastructure-as-Code (IaC) tool used to manage cloud infrastructure. In this tutorial, you'll use Pulumi to provision the AWS resources required for Kubernetes.

#### Create a New Pulumi Project

Run the following command to initialize a new Pulumi project:

```sh
pulumi new aws-python
```

Pulumi will guide you through setting up a new project and configuring it to use AWS resources.

#### Update the `__main.py__` file:

Update the __main.py__ file to create the necessary AWS infrastructure:
```python
import pulumi
import pulumi_aws as aws
import os

# Create a VPC
vpc = aws.ec2.Vpc(
    'kubernetes-vpc',
    cidr_block='10.0.0.0/16',
    enable_dns_support=True,
    enable_dns_hostnames=True,
    tags={'Name': 'kubernetes-the-hard-way'}
)

# Create a subnet
subnet = aws.ec2.Subnet(
    'kubernetes-subnet',
    vpc_id=vpc.id,
    cidr_block='10.0.1.0/24',
    map_public_ip_on_launch=True,
    tags={'Name': 'kubernetes'}
)

# Create an Internet Gateway
internet_gateway = aws.ec2.InternetGateway(
    'kubernetes-internet-gateway',
    vpc_id=vpc.id,
    tags={'Name': 'kubernetes'}
)

# Create a Route Table
route_table = aws.ec2.RouteTable(
    'kubernetes-route-table',
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block='0.0.0.0/0',
            gateway_id=internet_gateway.id,
        )
    ],
    tags={'Name': 'kubernetes'}
)

# Associate the route table with the subnet
route_table_association = aws.ec2.RouteTableAssociation(
    'kubernetes-route-table-association',
    subnet_id=subnet.id,
    route_table_id=route_table.id
)

# Create a security group with egress and ingress rules
security_group = aws.ec2.SecurityGroup(
    'kubernetes-security-group',
    vpc_id=vpc.id,
    description="Kubernetes security group",
    ingress=[
        aws.ec2.SecurityGroupIngressArgs(
            protocol='-1',
            from_port=0,
            to_port=0,
            cidr_blocks=['10.0.0.0/16', '10.200.0.0/16'],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol='tcp',
            from_port=22,
            to_port=22,
            cidr_blocks=['0.0.0.0/0'],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol='tcp',
            from_port=6443,
            to_port=6443,
            cidr_blocks=['0.0.0.0/0'],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol='tcp',
            from_port=443,
            to_port=443,
            cidr_blocks=['0.0.0.0/0'],
        ),
        aws.ec2.SecurityGroupIngressArgs(
            protocol='icmp',
            from_port=-1,
            to_port=-1,
            cidr_blocks=['0.0.0.0/0'],
        ),
    ],
    egress=[
        aws.ec2.SecurityGroupEgressArgs(
            protocol='-1',  # -1 allows all protocols
            from_port=0,
            to_port=0,
            cidr_blocks=['0.0.0.0/0'],  # Allow all outbound traffic
        )
    ],
    tags={'Name': 'kubernetes'}
)

# Create EC2 Instances for Controllers
controller_instances = []
for i in range(2):
    controller = aws.ec2.Instance(
        f'controller-{i}',
        instance_type='t2.small',
        ami='ami-01811d4912b4ccb26',  # Update with correct Ubuntu AMI ID
        subnet_id=subnet.id,
        key_name="kubernetes",
        vpc_security_group_ids=[security_group.id],
        associate_public_ip_address=True,
        private_ip=f'10.0.1.1{i}',
        tags={
            'Name': f'controller-{i}'
        }
    )
    controller_instances.append(controller)

# Create EC2 Instances for Workers
worker_instances = []
for i in range(2):
    worker = aws.ec2.Instance(
        f'worker-{i}',
        instance_type='t2.small',
        ami='ami-01811d4912b4ccb26',  # Update with correct Ubuntu AMI ID
        subnet_id=subnet.id,
        key_name="kubernetes",
        vpc_security_group_ids=[security_group.id],
        associate_public_ip_address=True,
        private_ip=f'10.0.1.2{i}',
        tags={'Name': f'worker-{i}'}
    )
    worker_instances.append(worker)

# Create a Network Load Balancer
nlb = aws.lb.LoadBalancer(
    'kubernetes-nlb',
    internal=False,
    load_balancer_type='network',
    subnets=[subnet.id],
    name='kubernetes'
)

# Create a Target Group for the Load Balancer
target_group = aws.lb.TargetGroup(
    'kubernetes-target-group',
    port=6443,
    protocol='TCP',
    vpc_id=vpc.id,
    target_type='ip',
    health_check=aws.lb.TargetGroupHealthCheckArgs(
        protocol='TCP',
    )
)

# Register Instances in Target Group
def create_attachment(name, target_id):
    return aws.lb.TargetGroupAttachment(
        name,
        target_group_arn=target_group.arn,
        target_id=target_id,
        port=6443
    )

# Iterate over controller instances and create TargetGroupAttachment
for i, instance in enumerate(controller_instances):
    # Use `apply` to get the resolved values of `instance.private_ip` and `instance.tags["Name"]`
    target_id = instance.private_ip
    attachment_name = instance.tags["Name"].apply(lambda tag_name: f'controller-{tag_name}-tg-attachment-{i}')
    
    # Ensure that `name` and `target_id` are resolved before creating the resource
    attachment = pulumi.Output.all(target_id, attachment_name).apply(lambda vals: create_attachment(vals[1], vals[0]))

    # Debug output
    pulumi.log.info(f'Creating TargetGroupAttachment with name: {attachment_name}')

# Create a Listener for the Load Balancer
listener = aws.lb.Listener(
    'kubernetes-listener',
    load_balancer_arn=nlb.arn,
    port=443,
    protocol='TCP',
    default_actions=[aws.lb.ListenerDefaultActionArgs(
        type='forward',
        target_group_arn=target_group.arn,
    )]
)

# Export Public DNS Name of the NLB
pulumi.export('kubernetes_public_address', nlb.dns_name)

# Export Public and Private IPs of Controller and Worker Instances
controller_public_ips = [controller.public_ip for controller in controller_instances]
controller_private_ips = [controller.private_ip for controller in controller_instances]
worker_public_ips = [worker.public_ip for worker in worker_instances]
worker_private_ips = [worker.private_ip for worker in worker_instances]

pulumi.export('controller_public_ips', controller_public_ips)
pulumi.export('controller_private_ips', controller_private_ips)
pulumi.export('worker_public_ips', worker_public_ips)
pulumi.export('worker_private_ips', worker_private_ips)

# Export the VPC ID and Subnet ID for reference
pulumi.export('vpc_id', vpc.id)
pulumi.export('subnet_id', subnet.id)

# create config file
def create_config_file(ip_list):
    # Define the hostnames for each IP address
    hostnames = ['controller-0', 'controller-1', 'worker-0', 'worker-1']
    
    config_content = ""
    
    # Iterate over IP addresses and corresponding hostnames
    for hostname, ip in zip(hostnames, ip_list):
        config_content += f"Host {hostname}\n"
        config_content += f"    HostName {ip}\n"
        config_content += f"    User ubuntu\n"
        config_content += f"    IdentityFile ~/.ssh/kubernetes.id_rsa\n\n"
    
    # Write the content to the SSH config file
    config_path = os.path.expanduser("~/.ssh/config")
    with open(config_path, "w") as config_file:
        config_file.write(config_content)

# Collect the IPs for all nodes
all_ips = [controller.public_ip for controller in controller_instances] + [worker.public_ip for worker in worker_instances]

# Create the config file with the IPs once the instances are ready
pulumi.Output.all(*all_ips).apply(create_config_file)
```

### Step 4: Create an AWS Key Pair

Kubernetes nodes need to communicate securely. This key pair will be used to authenticate when accessing EC2 instances.

#### Generate the Key Pair

Use the following AWS CLI command to create a new SSH key pair named `kubernetes`:

```sh
cd ~/.ssh/
aws ec2 create-key-pair --key-name kubernetes --output text --query 'KeyMaterial' > kubernetes.id_rsa
chmod 400 kubernetes.id_rsa
```

This will save the private key as `kubernetes.id_rsa` in your `~/.ssh/` directory and restrict its permissions.

## Export Kubernetes Public Address

After deploying your Kubernetes cluster, you’ll want to interact with it using `kubectl`. For this, you'll need the public address of the Kubernetes API server (typically the load balancer DNS).

### Step 1: Get the Load Balancer DNS Name

Run the following command to fetch the DNS name of the AWS load balancer that will be fronting your Kubernetes API:

```sh
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
export KUBERNETES_PUBLIC_ADDRESS
echo $KUBERNETES_PUBLIC_ADDRESS
```

> **Note**: The compute resources required for this tutorial exceed the free tier of Amazon Web Services (AWS). Ensure you clean up resources after completing this tutorial to avoid unnecessary costs.

### Step 2: Export Kubernetes Hostnames

These hostnames are used to reference your Kubernetes API server. Set them as an environment variable for later use:

```sh
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
export KUBERNETES_HOSTNAMES
echo $KUBERNETES_HOSTNAMES
```

---

This document sets up the basic infrastructure and client tools necessary for following Kubernetes the Hard Way. Each step ensures that you have the right environment, permissions, and tools to proceed further with setting up a Kubernetes cluster.

## SSH into the Instances

SSH into the instances, `controller-0`, controller-1, worker-0, worker-1 using these commands:

- **controller-0**
```sh
ssh controller-0
```
- **controller-1**
```sh
ssh controller-1
```
- **worker-0**
```sh
ssh worker-0
```
- **worker-0**
```sh
ssh worker-1
```