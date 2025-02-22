#!/bin/bash

# AWS Profile and Region input from the user
echo "Please provide your AWS Profile (leave empty to use 'default'):"
read AWS_PROFILE
if [ -z "$AWS_PROFILE" ]; then
  AWS_PROFILE="default"
fi

echo "Please provide your AWS Region (leave empty to use 'us-east-1'):"
read AWS_REGION
if [ -z "$AWS_REGION" ]; then
  AWS_REGION="us-east-1"
fi

# Logging setup
LOG_FILE="/var/log/nacl_cleanup.log"
touch $LOG_FILE
exec > >(tee -a $LOG_FILE)
exec 2>&1

echo "$(date) - Using AWS Profile: $AWS_PROFILE"
echo "$(date) - Using AWS Region: $AWS_REGION"

# List Network ACLs in the specified region
echo "$(date) - Fetching Network ACLs in region $AWS_REGION..."
NACLS=$(aws ec2 describe-network-acls --profile $AWS_PROFILE --region $AWS_REGION --query 'NetworkAcls[].[NetworkAclId]' --output text)

if [ -z "$NACLS" ]; then
  echo "No Network ACLs found."
  exit 0
fi

# Loop through each Network ACL and check if it has the "BotIgnore" tag
for NACL in $NACLS; do
  echo "$(date) - Checking NACL: $NACL"

  # Check for the "BotIgnore" tag with value "True"
  TAGS=$(aws ec2 describe-tags --resources $NACL --profile $AWS_PROFILE --region $AWS_REGION --query 'Tags[?Key==`BotIgnore`].Value' --output text)

  if [ "$TAGS" == "True" ]; then
    echo "$(date) - NACL $NACL has 'BotIgnore' tag with value 'True'. Skipping deletion."
  else
    echo "$(date) - NACL $NACL does not have 'BotIgnore' tag or has a different value. Proceeding with deletion."

    # Confirmation prompt before deletion
    read -p "Do you want to delete this Network ACL ($NACL)? [y/N]: " confirm
    if [[ $confirm == [Yy] ]]; then
      # Deleting the Network ACL
      aws ec2 delete-network-acl --network-acl-id $NACL --profile $AWS_PROFILE --region $AWS_REGION
      echo "$(date) - Deleted Network ACL: $NACL"
    else
      echo "$(date) - Skipping Network ACL: $NACL"
    fi
  fi
done

echo "$(date) - Network ACL cleanup script finished."
