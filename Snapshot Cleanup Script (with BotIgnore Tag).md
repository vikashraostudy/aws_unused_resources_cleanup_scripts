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
LOG_FILE="/var/log/snapshot_cleanup.log"
touch $LOG_FILE
exec > >(tee -a $LOG_FILE)
exec 2>&1

echo "$(date) - Using AWS Profile: $AWS_PROFILE"
echo "$(date) - Using AWS Region: $AWS_REGION"

# List Snapshots in the specified region
echo "$(date) - Fetching Snapshots in region $AWS_REGION..."
SNAPSHOTS=$(aws ec2 describe-snapshots --profile $AWS_PROFILE --region $AWS_REGION --query 'Snapshots[].[SnapshotId]' --output text)

if [ -z "$SNAPSHOTS" ]; then
  echo "No Snapshots found."
  exit 0
fi

# Loop through each Snapshot and check if it has the "BotIgnore" tag
for SNAPSHOT in $SNAPSHOTS; do
  echo "$(date) - Checking Snapshot: $SNAPSHOT"

  # Check for the "BotIgnore" tag with value "True"
  TAGS=$(aws ec2 describe-tags --resources $SNAPSHOT --profile $AWS_PROFILE --region $AWS_REGION --query 'Tags[?Key==`BotIgnore`].Value' --output text)

  if [ "$TAGS" == "True" ]; then
    echo "$(date) - Snapshot $SNAPSHOT has 'BotIgnore' tag with value 'True'. Skipping deletion."
  else
    echo "$(date) - Snapshot $SNAPSHOT does not have 'BotIgnore' tag or has different value. Proceeding with deletion."

    # Confirmation prompt before deletion
    read -p "Do you want to delete this Snapshot ($SNAPSHOT)? [y/N]: " confirm
    if [[ $confirm == [Yy] ]]; then
      # Deleting the Snapshot
      aws ec2 delete-snapshot --snapshot-id $SNAPSHOT --profile $AWS_PROFILE --region $AWS_REGION
      echo "$(date) - Deleted Snapshot: $SNAPSHOT"
    else
      echo "$(date) - Skipping Snapshot: $SNAPSHOT"
    fi
  fi
done

echo "$(date) - Snapshot cleanup script finished."
