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
LOG_FILE="/var/log/ami_cleanup.log"
touch $LOG_FILE
exec > >(tee -a $LOG_FILE)
exec 2>&1

echo "$(date) - Using AWS Profile: $AWS_PROFILE"
echo "$(date) - Using AWS Region: $AWS_REGION"

# List AMIs in the specified region
echo "$(date) - Fetching AMIs in region $AWS_REGION..."
AMIS=$(aws ec2 describe-images --profile $AWS_PROFILE --region $AWS_REGION --query 'Images[].[ImageId]' --output text)

if [ -z "$AMIS" ]; then
  echo "No AMIs found."
  exit 0
fi

# Loop through each AMI and check if it has the "BotIgnore" tag
for AMI in $AMIS; do
  echo "$(date) - Checking AMI: $AMI"

  # Check for the "BotIgnore" tag with value "True"
  TAGS=$(aws ec2 describe-tags --resources $AMI --profile $AWS_PROFILE --region $AWS_REGION --query 'Tags[?Key==`BotIgnore`].Value' --output text)

  if [ "$TAGS" == "True" ]; then
    echo "$(date) - AMI $AMI has 'BotIgnore' tag with value 'True'. Skipping deletion."
  else
    echo "$(date) - AMI $AMI does not have 'BotIgnore' tag or has different value. Proceeding with deletion."

    # Confirmation prompt before deletion
    read -p "Do you want to deregister this AMI ($AMI)? [y/N]: " confirm
    if [[ $confirm == [Yy] ]]; then
      # Deregistering the AMI
      aws ec2 deregister-image --image-id $AMI --profile $AWS_PROFILE --region $AWS_REGION
      echo "$(date) - Deregistered AMI: $AMI"
    else
      echo "$(date) - Skipping AMI: $AMI"
    fi
  fi
done

echo "$(date) - AMI cleanup script finished."
