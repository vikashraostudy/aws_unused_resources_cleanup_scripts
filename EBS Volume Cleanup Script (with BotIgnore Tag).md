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
LOG_FILE="/var/log/ebs_cleanup.log"
touch $LOG_FILE
exec > >(tee -a $LOG_FILE)
exec 2>&1

echo "$(date) - Using AWS Profile: $AWS_PROFILE"
echo "$(date) - Using AWS Region: $AWS_REGION"

# List EBS Volumes in the specified region
echo "$(date) - Fetching EBS Volumes in region $AWS_REGION..."
VOLUMES=$(aws ec2 describe-volumes --profile $AWS_PROFILE --region $AWS_REGION --query 'Volumes[].[VolumeId, State]' --output text)

if [ -z "$VOLUMES" ]; then
  echo "No EBS volumes found."
  exit 0
fi

# Loop through each EBS Volume and check if it has the "BotIgnore" tag
for VOLUME in $VOLUMES; do
  VOLUME_ID=$(echo $VOLUME | awk '{print $1}')
  STATE=$(echo $VOLUME | awk '{print $2}')

  echo "$(date) - Checking EBS Volume: $VOLUME_ID with state: $STATE"

  # Check for the "BotIgnore" tag with value "True"
  TAGS=$(aws ec2 describe-tags --resources $VOLUME_ID --profile $AWS_PROFILE --region $AWS_REGION --query 'Tags[?Key==`BotIgnore`].Value' --output text)

  if [ "$TAGS" == "True" ]; then
    echo "$(date) - EBS Volume $VOLUME_ID has 'BotIgnore' tag with value 'True'. Skipping deletion."
  else
    echo "$(date) - EBS Volume $VOLUME_ID does not have 'BotIgnore' tag or has a different value. Proceeding with deletion."

    # Check if the EBS volume is attached to any instance
    ATTACHED_INSTANCE=$(aws ec2 describe-volumes --volume-ids $VOLUME_ID --profile $AWS_PROFILE --region $AWS_REGION --query 'Volumes[0].Attachments[0].InstanceId' --output text)

    if [ "$ATTACHED_INSTANCE" == "None" ]; then
      # Volume is not attached, safe to delete
      echo "$(date) - EBS Volume $VOLUME_ID is not attached to any instance. Proceeding with deletion."
      read -p "Do you want to delete EBS Volume $VOLUME_ID? [y/N]: " confirm
      if [[ $confirm == [Yy] ]]; then
        # Deleting the EBS Volume
        aws ec2 delete-volume --volume-id $VOLUME_ID --profile $AWS_PROFILE --region $AWS_REGION
        echo "$(date) - Deleted EBS Volume: $VOLUME_ID"
      else
        echo "$(date) - Skipping deletion of EBS Volume: $VOLUME_ID"
      fi
    else
      echo "$(date) - EBS Volume $VOLUME_ID is still attached to an instance. Skipping deletion."
    fi
  fi
done

echo "$(date) - EBS Volume cleanup script finished."
