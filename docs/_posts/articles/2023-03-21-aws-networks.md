---
layout: post
title:  "AWS Deployment Part I"
categories: posts
subtitle: AWS Deployment Part I, the network
accent_image: /assets/img/blog/aws-p1/banner.png
accent_color: '#FA8D22'
image:
  path:    /assets/img/blog/aws-p1/banner.png
  srcset:
    1920w: /assets/img/blog/aws-p1/banner.png
---

## Introduction

I have recently made the decision to expand my skills in AWS. To achieve this goal, I have chosen to deploy a number of useful resources on the platform using Terraform. My initial objective is to develop a basic instance deployment that is modular and flexible, allowing it to be easily reused in the future. This will be a great foundation for more advanced deployments in the future.

In addition to deploying the resources, I will also be incorporating tests into the code. This will ensure that the code is easier to maintain and that any issues can be quickly identified and resolved. By taking these steps, I believe that I will be well on my way to developing a solid foundation of skills in AWS, which will serve me well in the future.

I would also like to take this opportunity to test some new tools that could help me with this task:

- [Chatcody](https://github.com/marketplace/chatcody): Since I'm working on this project alone, I don't have anyone to review my code. This AI-based tool can "replace" a colleague and provide feedback on the code I push.
- [ChatGPT](https://ai.com): This tool is widely discussed, and there are better articles available, so I won't write much about it. However, I will use it to assist me with this exercise.

![Full-width image](/assets/img/blog/aws-p1/schema.png){:.lead width="800" height="100" loading="lazy"}
Basic schema of what we want to achieve
{:.figcaption}

- Network
    - 1 VPC
    - 3 subnets
    - 1 Security Group
    - 1 Internet Gateway & routes
- Computing
    - 1 Auto Scaling Group
    - 1 Load Balancer
- Logs & Metrics
    - CloudWatch

## Basic Network

In order to deploy ASG instances on AWS, you'll first need to set up a basic network. This includes creating a VPC, subnets, security groups, routes an an Internet Gateway.  
**We will focus on this point on this article.**  

Currently, I am keeping this repository public. You can find it [here](https://github.com/DucretJe/std-deploy).

### VPC

Let’s start simple, we want a single VPC, and we want to be able to choose the CIDR in the future:

```terraform
resource "aws_vpc" "this" {
  cidr_block = var.vpc_cidr

  tags = var.vpc_tags
}
```

We have to declare the variables, and this is it! Ready for our first commit 🚀  
It's not quite ready yet. As I mentioned earlier, we want to add some tests to make it easier to maintain in the future. Although it may seem like overkill at the moment, since it's currently quite simple, adding tests will be a good way to train ourselves and hopefully prevent accidental errors in the future.

To test the deployment, we will add some Python tests to check if the resource is available.

Since I was unsure how to test it in Python, I asked ChatGPT for assistance. I tsuggested using the AWS SDK for Python (`Boto3`) library and helped me create the following function:

```python
def test_vpc_exists(region_name, vpc_id):
    session = boto3.Session()

    ec2_client = session.client("ec2", region_name=region_name)

    try:
        response = ec2_client.describe_vpcs(VpcIds=[vpc_id])
    except ec2_client.exceptions.ClientError as e:
        if e.response["Error"]["Code"] == "InvalidVpcID.NotFound":
            print("VPC {vpc_id} does not exist")
            assert False, f"VPC {vpc_id} does not exist"
        else:
            print(f"An error occurred: {e}")
            assert False, f"An error occurred: {e}"
```

Assuming that we are provided with credentials and a region, the first step is to create a session and describe the VPC provided as an argument. If the description fails, the test fails as well, indicating that there is an error somewhere.

Although it may not be particularly useful, because it is essentially what Terraform does when creating a resource (waiting for the resource to exist and retrieving its ID), we will perform some more complex tests later.

To simplify the testing process, we can create a `tests` directory that includes our Python code, makefile, and a Terraform file that calls our new network module.

![Full-width image](/assets/img/blog/aws-p1/folder.png){:.lead width="800" height="100" loading="lazy"}
Our `tests` directory
{:.figcaption}

Our `main.tf` file is quite simple

```terraform
module "network" {
  source = "../../aws/"

  vpc_cidr = "10.0.0.0/16"
  vpc_tags = {
    terraform = "true"
  }
}
```

In order to make our test work we need to provide the created VPC’s ID so we have to add an output to our module and in our `main.tf`.

Our `MakeFile` will contain the following:

```makefile
.PHONY: all plan

all: test

plan:
	cd 01-network && \
	terraform init && \
	terraform validate && \
	terraform plan  && \
	echo "01-network: SHOULD BE OK"

test:
	cd 01-network && \
	terraform init && \
	terraform validate && \
	terraform apply --auto-approve && \
	cd .. && \
	python3 -m venv .venv && \
	. .venv/bin/activate && \
	pip install -r requirements.txt && \
	python3 test.py --region ${AWS_REGION} --vpc-id $$(cd 01-network && terraform output vpc_id | sed 's/"//g') && \
	cd 01-network && \
	terraform destroy --auto-approve && \
	echo "01-network: OK"

destroy:
	cd 01-network && \
	terraform destroy --auto-approve
```

Finally we add a workflow to automate the tests, also we finish our Python test to handle the arguments and give it a `main` and we’re good to go! 💪  

Well… not so fast, `checkov` is not so happy with our terraform code, since we don’t log anything. So reading a bit its error we need to:

- Create a role for the `vpc-logs`
- A policy allowing it to create logs and logs groups, and actually push logs
- Another policy allowing the service [`vpc-flow-logs.amazonaws.com`](http://vpc-flow-logs.amazonaws.com/) to assume our `vpc-logs` role
- A log group in CloudWatch
- A log flow

This makes our `vpc.tf` file a bit longer since we add:

```terraform
resource "aws_flow_log" "vpc_logs" {
  iam_role_arn    = aws_iam_role.vpc_logs.arn
  log_destination = aws_cloudwatch_log_group.vpc_logs.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.this.id

  tags = var.vpc_tags
}

resource "aws_cloudwatch_log_group" "vpc_logs" {
  name = "vpc-logs"
  retention_in_days = 30

  tags = var.vpc_tags
}

data "aws_iam_policy_document" "assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["vpc-flow-logs.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "vpc_logs" {
  name               = "vpc-logs"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json

  tags = var.vpc_tags
}

data "aws_iam_policy_document" "vpc_logs" {
  statement {
    effect = "Allow"

    actions = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams",
    ]

    resources = ["*"]
  }
}

resource "aws_iam_role_policy" "vpc_logs" {
  name   = "vpc-logs"
  role   = aws_iam_role.vpc_logs.id
  policy = data.aws_iam_policy_document.vpc_logs.json
}
```

### Subnets

Ok, we now have a VPC, but it is currently empty. Let's populate it by adding some subnets.

To ensure our module is as easy to use as possible, we have already declared a CIDR for our VPC. In our test, the CIDR is `10.0.0.0/16`. Our objective is to create subnets in all zones without having to manually calculate and declare their CIDR.

```terraform
data "aws_availability_zones" "all" {}

locals {
  subnet_cidr_blocks = [for i in range(length(data.aws_availability_zones.all.names)) :
    cidrsubnet(aws_vpc.this.cidr_block, 8, i)
  ]
}

resource "aws_subnet" "this" {
  count = length(data.aws_availability_zones.all.names)

  vpc_id            = aws_vpc.this.id
  cidr_block        = local.subnet_cidr_blocks[count.index]
  availability_zone = data.aws_availability_zones.all.names[count.index]
}
```

The idea is to detect the zones where we can deploy (the region is set up by the provider parameters), and we use the `aws_availability_zones` data for this purpose.

We perform some magic in the `locals` 🪄

We ask Terraform to repeat the calculation for each availability zone and then calculate a CIDR with an 8-bit mask shift. This way, we get different CIDRs. For example, if we deploy it in a VPC with three availability zones, we will get:

- `10.0.0.0/24`
- `10.0.1.0/24`
- `10.0.2.0/24`

We'll receive 251 free IPs per subnet. This is sufficient for our project, but it would be an improvement for our module to allow users to choose the size of their subnets. 💪  

Our continuous integration and `Chatcody` are satisfied. This step is ready to be merged!

### Security Group

Connectivity is important, but we must also have control over it.

_Power is nothing without control_
{:.note title="💭 Wisdom"}

To control the traffic that flows through our subnets, we need to add security groups.

```terraform
resource "aws_security_group" "this" {
  name_prefix = var.sg_name
  description = var.sg_description
  vpc_id      = aws_vpc.this.id

  dynamic "ingress" {
    for_each = var.sg_ingress_rules
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  dynamic "egress" {
    for_each = var.sg_egress_rules
    content {
      description = egress.value.description
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }
}
```

As we cannot know in advance the rules we will set, we have set some dynamic blocks to allow us to do whatever we want in our security group.

To test it, we will set the following variables in our `module.network`:

```terraform
sg_description = "Security group for the VPC"
  sg_egress_rules = [
    {
      description = "Allow all outbound traffic"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
  sg_ingress_rules = [
    {
      description = "Allow HTTPS inbound traffic"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
  sg_name = "sg"
```

I agree that a *yolo* rule for egress is not ideal. However, since the test deployment is aimed to be ephemeral, it may not be a big deal, and it could potentially help with future debugging. Therefore, let's agree to keep it this way.

Well, that's good enough for me. Maybe we can improve by finding a more efficient way to spawn security groups. For example, we could develop a method to create several security groups simultaneously.

### Internet Gateway

The network setup is almost complete! My use case involves creating an instance with a random web server, testing it with a GET request, and expecting a 200 code response. In order to do this from the GitHub Action runner, the instance needs to be reachable over the internet. Therefore, we need to set up an internet gateway.

I'm not sure if I need it every time, so I wanted to make it optional.

```terraform
resource "aws_internet_gateway" "this" {
  count  = var.internet_gateway ? 1 : 0
  vpc_id = aws_vpc.this.id

  tags = var.internet_gateway_tags
}

resource "aws_route_table" "my_rt" {
  count  = var.internet_gateway ? 1 : 0
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this[0].id
  }

  tags = var.route_table_tags
}

resource "aws_route_table_association" "this" {
  count          = length(aws_subnet.this)
  subnet_id      = aws_subnet.this[count.index].id
  route_table_id = aws_route_table.my_rt[0].id
}
```

To connect two resources, we create a third one that acts as glue. In this case, we create the following:

- The gateway, which uses the VPC we created and to which we can add tags.
- The route table, which aims at the VPC we created and adds a default route aimed at the gateway. We can also add tags to the route table.
- The route table association, which associates each created subnet to the route table.

`var.internet_gateway` is set to `true` by default, so we don't need to declare variables to use it.

### Enhance our tests

The final section of this blog post will cover Python tests. While we have laid some groundwork, there is more work to be done to improve the tests. It is our goal to test all of our resources thoroughly.

- VPC
- Security Group
- Subnets
- Internet Gateway

We want all items to be tested and avoid the script from stopping on the first issue. To achieve this, we create a new class called `TestFailed` and use it in the function `run_test`. This function runs our different tests as a list and returns their exception, if any. We want it to run all tests and report their results afterwards.

```python
import argparse

import boto3

def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--region", required=True, help="AWS region")
    parser.add_argument("--vpc-id", required=True, help="VPC ID")
    parser.add_argument("--security-group-id", required=True, help="Security group ID")
    parser.add_argument(
        "--subnet-ids",
        required=True,
        nargs="+",
        help="List of subnet IDs",
    )
    return parser.parse_args()

class TestFailed(Exception):
    pass

def test_vpc_exists(region_name, vpc_id):
    try:
        session = boto3.Session()
        ec2_client = session.client("ec2", region_name=region_name)
        response = ec2_client.describe_vpcs(VpcIds=[vpc_id])
        assert len(response["Vpcs"]) == 1, f"VPC {vpc_id} does not exist"
        print(f"VPC {vpc_id} exists")
    except Exception as e:
        raise TestFailed(f"Failed to test VPC {vpc_id}: {e}")

def test_security_group_exists(region_name, security_group_id):
    try:
        session = boto3.Session()
        ec2_client = session.client("ec2", region_name=region_name)
        response = ec2_client.describe_security_groups(GroupIds=[security_group_id])
        assert (
            len(response["SecurityGroups"]) == 1
        ), f"Security group {security_group_id} does not exist"
        print(f"Security group {security_group_id} exists")
    except Exception as e:
        raise TestFailed(f"Failed to test security group {security_group_id}: {e}")

def test_subnets_exist(region_name, vpc_id, subnet_ids):
    try:
        session = boto3.Session()
        ec2_client = session.client("ec2", region_name=region_name)
        response = ec2_client.describe_subnets(SubnetIds=subnet_ids)
        subnet_ids_in_aws = [subnet["SubnetId"] for subnet in response["Subnets"]]
        assert len(subnet_ids) == len(subnet_ids_in_aws), "Not all subnets exist"
        print("All subnets exist")
    except Exception as e:
        raise TestFailed(f"Failed to test subnets: {e}")

def test_internet_gateway_exists(region_name, vpc_id):
    try:
        session = boto3.Session()
        ec2_client = session.client("ec2", region_name=region_name)
        response = ec2_client.describe_internet_gateways(
            Filters=[{"Name": "attachment.vpc-id", "Values": [vpc_id]}]
        )
        assert (
            len(response["InternetGateways"]) == 1
        ), f"Internet Gateway does not exist for VPC {vpc_id}"
        print(f"Internet Gateway exists for VPC {vpc_id}")
    except Exception as e:
        raise TestFailed(f"Failed to test Internet Gateway: {e}")

def run_test(test_func, test_args):
    try:
        test_func(*test_args)
    except TestFailed as e:
        print(f"ERROR: {e}")
        return False
    return True

if __name__ == "__main__":
    args = parse_args()

    test_cases = [
        (test_vpc_exists, [args.region, args.vpc_id]),
        (test_security_group_exists, [args.region, args.security_group_id]),
        (test_subnets_exist, [args.region, args.vpc_id, args.subnet_ids]),
        (test_internet_gateway_exists, [args.region, args.vpc_id]),
    ]

    has_errors = False
    for test_func, test_args in test_cases:
        if not run_test(test_func, test_args):
            has_errors = True
```

### CICD

We have our Terraform files, tests, and a Makefile to run them. The only thing missing is the CI to execute the `make all` command. We also want it to destroy everything created, regardless of the test results. This is crucial to avoid retaining unnecessary resources in our AWS region.

We want the process to run automatically on every pull request that modifies the code, but we also need the ability to trigger it manually if necessary.

```yaml
---
name: "🔬 Tests"

on:
  pull_request:
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform.yaml'
  workflow_dispatch: # Allow to manually trigger the pipeline

  network:
    name: 🕸️ Network
    runs-on: ubuntu-22.04
    steps:
      - name: 🛎️ Checkout
        uses: actions/checkout@v3

      - name: 🚀 Make Plan
        run: make all
        working-directory: terraform/network/tests
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: eu-central-1

      - name: 💥 Make Destroy
        run: make destroy
        if: always()
        working-directory: terraform/network/tests
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: eu-central-1
```

Finally, with these steps complete, we can move on to the computing module in the next part.
