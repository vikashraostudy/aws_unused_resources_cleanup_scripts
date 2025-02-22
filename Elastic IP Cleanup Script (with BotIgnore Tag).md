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
LOG_FILE="/var/log/eip_cleanup.log"
touch $LOG_FILE
exec > >(tee -a $LOG_FILE)
exec 2>&1

echo "$(date) - Using AWS Profile: $AWS_PROFILE"
echo "$(date) - Using AWS Region: $AWS_REGION"

# List Elastic IPs in the specified region
echo "$(date) - Fetching Elastic IPs in region $AWS_REGION..."
EIPS=$(aws ec2 describe-addresses --profile $AWS_PROFILE --region $AWS_REGION --query 'Addresses[].[AllocationId, PublicIp]' --output text)

if [ -z "$EIPS" ]; then
  echo "No Elastic IPs found."
  exit 0
fi

# Loop through each Elastic IP and check if it has the "BotIgnore" tag
for EIP in $EIPS; do
  ALLOCATION_ID=$(echo $EIP | awk '{print $1}')
  PUBLIC_IP=$(echo $EIP | awk '{print $2}')

  echo "$(date) - Checking Elastic IP: $PUBLIC_IP ($ALLOCATION_ID)"

  # Check for the "BotIgnore" tag with value "True"
  TAGS=$(aws ec2 describe-tags --resources $ALLOCATION_ID --profile $AWS_PROFILE --region $AWS_REGION --query 'Tags[?Key==`BotIgnore`].Value' --output text)

  if [ "$TAGS" == "True" ]; then
    echo "$(date) - Elastic IP $PUBLIC_IP ($ALLOCATION_ID) has 'BotIgnore' tag with value 'True'. Skipping release."
  else
    echo "$(date) - Elastic IP $PUBLIC_IP ($ALLOCATION_ID) does not have 'BotIgnore' tag or has a different value. Proceeding with release."

    # Check if the Elastic IP is associated with any resource
    ASSOCIATION_ID=$(aws ec2 describe-addresses --allocation-ids $ALLOCATION_ID --profile $AWS_PROFILE --region $AWS_REGION --query 'Addresses[0].AssociationId' --output text)

    if [ "$ASSOCIATION_ID" == "None" ]; then
      # No association, safe to release
      echo "$(date) - Elastic IP $PUBLIC_IP is not associated with any resource. Proceeding with release."
      read -p "Do you want to release Elastic IP $PUBLIC_IP ($ALLOCATION_ID)? [y/N]: " confirm
      if [[ $confirm == [Yy] ]]; then
        # Releasing the Elastic IP
        aws ec2 release-address --allocation-id $ALLOCATION_ID --profile $AWS_PROFILE --region $AWS_REGION
        echo "$(date) - Released Elastic IP: $PUBLIC_IP ($ALLOCATION_ID)"
      else
        echo "$(date) - Skipping release of Elastic IP: $PUBLIC_IP"
      fi
    else
      echo "$(date) - Elastic IP $PUBLIC_IP ($ALLOCATION_ID) is still associated with a resource. Skipping release."
    fi
  fi
done

echo "$(date) - Elastic IP cleanup script finished."
