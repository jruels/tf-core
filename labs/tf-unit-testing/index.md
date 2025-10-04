# Terraform Unit Testing

## Overview

In this lab, you will add unit tests to your existing `learn-terraform-variables` Terraform configuration to validate configuration logic, variable validation, and conditional expressions without creating actual infrastructure. You'll test the existing VPC, security groups, load balancer, and EC2 instance modules to ensure they behave correctly under different input scenarios.

## Prerequisites

This lab builds upon the previous labs. You should have completed:
- `hcp-tf-setup` - HCP Terraform authentication and workspace setup
- `hcp-tf-modify` - Infrastructure modification and VCS workflow setup
- `hcp-tf-publish-module` - Module publishing and consumption

You should be working in your forked `learn-terraform-variables` repository with the existing infrastructure configuration.

## Understanding Your Current Configuration

Your existing configuration includes:
- **VPC Module**: Creates public and private subnets with NAT gateway
- **Security Groups**: Web server and load balancer security groups
- **Load Balancer**: Application Load Balancer with health checks
- **EC2 Instances**: Custom module creating web servers in private subnets
- **Variables**: `instance_count` and `instance_type` for EC2 instances

## Prepare Your Configuration for Testing

### 1. Remove AWS Credentials

If you have AWS credentials configured using `aws configure`, remove them for this lab:

```sh
rm ~/.aws/credentials
rm ~/.aws/config
```

### 2. Fix Provider Version Compatibility

First, update your `main.tf` to pin the AWS provider to a compatible version that works with the VPC module:

```hcl
terraform {
  # cloud {
  #   organization = "sudo-cloud-org"
  #   workspaces {
  #     name = "tf-vault-qa-{your-initials}"
  #   }
  # }
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.76.0"
    }
  }
}
```

### 3. Comment Out HCP Terraform Module

Since you're running unit tests locally, you don't have access to the HCP Terraform private registry. Comment out the S3 bucket module call in your `main.tf`:

```hcl
# module "s3_bucket" {
#   source  = "app.terraform.io/sudo-cloud-org/s3-bucket/aws"
#   version = "1.0.0"
#   bucket_name = "my-bucket"
# }
```

### 4. Initialize Terraform

Run terraform init to download the correct provider version:

```sh
terraform init -upgrade
```

## Create Unit Tests for Your Configuration

### 5. Create Test Directory

In your `learn-terraform-variables` repository, create a new directory for tests:

```sh
mkdir tests
cd tests
```

### 6. Test EC2 Instance Module

Create a test file `ec2_instance_tests.tftest.hcl` to validate your custom EC2 instance module:

```hcl
# tests/ec2_instance_tests.tftest.hcl

variables {
  instance_count = 2
  instance_type  = "t2.micro"
}

run "test_ec2_instance_count" {
  command = plan
  
  assert {
    condition     = length(module.ec2_instances.instance_ids) == 2
    error_message = "Should create exactly 2 EC2 instances"
  }
}
```

### 7. Test Variable Validation

Create a test file `variable_validation_tests.tftest.hcl` to test variable constraints:

```hcl
# tests/variable_validation_tests.tftest.hcl

run "test_valid_instance_count" {
  command = plan
  
  variables {
    instance_count = 3
    instance_type  = "t2.small"
  }
  
  # This should pass - valid instance count and type
}
```

## Run Your Unit Tests

1. From your `learn-terraform-variables` directory, run the tests:

```sh
terraform test tests/
```

**Expected Result: FAILURE** - You should see errors related to missing AWS credentials. This is expected because we haven't set up mocking yet.

## Fix the AWS Provider Issues

The tests are failing because they need AWS credentials. To fix this, add the following to the top of each test file:

```hcl
# Mock the AWS provider to avoid credential issues
mock_provider "aws" {}
```

2. Run the tests again:

```sh
terraform test tests/
```

**Expected Result: FAILURE** - You should now see errors related to empty availability zones data. This is expected because the data source needs to be mocked.

## Fix the Data Source Issues

To fix the availability zones errors, add the following to each test file after the provider mock:

```hcl
# Mock the availability zones data source
override_data {
  target = data.aws_availability_zones.available
  values = {
    names = ["us-west-1a", "us-west-1b", "us-west-1c"]
  }
}
```

3. Run the tests again:

```sh
terraform test tests/
```

This will execute all your test files and validate that your configuration logic works correctly for different input scenarios.

## Expected Results After Fixing

- All unit tests pass, validating your existing infrastructure configuration
- The correct number of EC2 instances are created
- Variable changes affect infrastructure as expected

## Reflection

- **Reflection:** How do unit tests complement the infrastructure testing you've done in previous labs? What are the trade-offs between testing speed and coverage?

## Next Steps

After completing this lab, you'll be ready to create integration tests that validate actual infrastructure creation and resource properties. Integration tests will build upon these unit testing concepts while adding real infrastructure validation. 