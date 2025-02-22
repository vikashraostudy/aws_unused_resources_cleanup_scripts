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
LOG_FILE="/var/log/security_group_cleanup.log"
touch $LOG_FILE
exec > >(tee -a $LOG_FILE)
exec 2>&1

echo "$(date) - Using AWS Profile: $AWS_PROFILE"
echo "$(date) - Using AWS Region: $AWS_REGION"

# List Security Groups in the specified region
echo "$(date) - Fetching Security Groups in region $AWS_REGION..."
SECURITY_GROUPS=$(aws ec2 describe-security-groups --profile $AWS_PROFILE --region $AWS_REGION --query 'SecurityGroups[].[GroupId, GroupName]' --output text)

if [ -z "$SECURITY_GROUPS" ]; then
  echo "No Security Groups found."
  exit 0
fi

# Loop through each Security Group and check if it has the "BotIgnore" tag
for SG in $SECURITY_GROUPS; do
  SG_ID=$(echo $SG | awk '{print $1}')
  SG_NAME=$(echo $SG | awk '{print $2}')

  echo "$(date) - Checking Security Group: $SG_ID - $SG_NAME"

  # Check for the "BotIgnore" tag with value "True"
  TAGS=$(aws ec2 describe-tags --resources $SG_ID --profile $AWS_PROFILE --region $AWS_REGION --query 'Tags[?Key==`BotIgnore`].Value' --output text)

  if [ "$TAGS" == "True" ]; then
    echo "$(date) - Security Group $SG_ID ($SG_NAME) has 'BotIgnore' tag with value 'True'. Skipping deletion."
  else
    echo "$(date) - Security Group $SG_ID ($SG_NAME) does not have 'BotIgnore' tag or has a different value. Proceeding with deletion."

    # Confirmation prompt before deletion
    read -p "Do you want to delete this Security Group ($SG_ID)? [y/N]: " confirm
    if [[ $confirm == [Yy] ]]; then
      # Deleting the Security Group
      aws ec2 delete-security-group --group-id $SG_ID --profile $AWS_PROFILE --region $AWS_REGION
      echo "$(date) - Deleted Security Group: $SG_ID"
    else
      echo "$(date) - Skipping Security Group: $SG_ID"
    fi
  fi
done

echo "$(date) - Security Group cleanup script finished."
