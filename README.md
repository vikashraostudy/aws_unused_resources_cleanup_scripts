# AWS Unused Resources Cleanup Scripts

This repository contains a set of scripts to help identify and clean up unused AWS resources, optimizing your AWS infrastructure and reducing unnecessary costs.

## Features

- Automated cleanup of unused EC2 instances, EBS volumes, S3 buckets, RDS databases, and more.
- Configurable scripts to target specific resources or regions.
- Works with AWS CLI and boto3 for automation and script-based management.
- Support for dry-run mode to preview changes before actually deleting resources.

## Requirements

- AWS CLI installed and configured.
- Python 3.6+ with `boto3` and `botocore` libraries installed.
- IAM roles with necessary permissions (EC2, S3, RDS, etc.).
  
You can install the required Python libraries using:
```bash
pip install boto3 botocore
