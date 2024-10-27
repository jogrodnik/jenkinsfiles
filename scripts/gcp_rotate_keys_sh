#!/bin/bash

set -e  # Exit on any error
set -o pipefail  # Catch any errors in pipelines

# Function to log messages with timestamps for better tracking
log_message() {
    echo "$(date +'%Y-%m-%d %H:%M:%S') - $1"
}

# Environment variables, which Jenkins should pass as parameters
PROJECT_ID="${PROJECT_ID:?Please set PROJECT_ID as an environment variable}"
BUCKET_NAME="${BUCKET_NAME:?Please set BUCKET_NAME as an environment variable}"
KEY_ADMIN="${KEY_ADMIN:?Please set KEY_ADMIN service account}"
SERVICE_ACCOUNTS="${SERVICE_ACCOUNTS:-"service-account-1@${PROJECT_ID}.iam.gserviceaccount.com,service-account-2@${PROJECT_ID}.iam.gserviceaccount.com,$KEY_ADMIN"}"

# Convert SERVICE_ACCOUNTS to an array
IFS=',' read -r -a SERVICE_ACCOUNTS_ARRAY <<< "$SERVICE_ACCOUNTS"

# Function to rotate keys for a service account
rotate_key() {
    local service_account=$1
    log_message "Starting key rotation for $service_account"

    # Step 1: Generate a new key and save it temporarily
    local new_key
    new_key=$(gcloud iam service-accounts keys create \
              --iam-account="$service_account" \
              --project="$PROJECT_ID" -q \
              --format="json" | jq -r .privateKeyData | base64 --decode)

    local temp_key_file="/tmp/${service_account//[@.]/_}_key.json"
    echo "$new_key" > "$temp_key_file"

    # Step 2: Upload the new key to the GCS bucket
    local gcs_key_path="gs://${BUCKET_NAME}/${service_account}_$(date +%Y%m%d).json"
    gsutil cp "$temp_key_file" "$gcs_key_path"
    log_message "Uploaded new key to $gcs_key_path"

    # For the key-admin service account, test new key access before deleting the old one
    if [[ "$service_account" == "$KEY_ADMIN" ]]; then
        log_message "Testing new key access for key-admin service account"

        # Activate the new key temporarily
        gcloud auth activate-service-account "$service_account" --key-file="$temp_key_file" --project="$PROJECT_ID"

        # Verify new key has access by listing keys (or other lightweight command)
        if gcloud iam service-accounts keys list --iam-account="$service_account" --project="$PROJECT_ID" > /dev/null 2>&1; then
            log_message "New key validated for key-admin service account"
        else
            log_message "Failed to verify new key for key-admin; aborting rotation for this account."
            rm "$temp_key_file"
            return 1
        fi
    fi

    # Step 3: Remove the oldest key to maintain rotation
    local key_ids
    key_ids=($(gcloud iam service-accounts keys list \
                --iam-account="$service_account" \
                --project="$PROJECT_ID" \
                --managed-by="user" \
                --format="value(name)"))

    if [ ${#key_ids[@]} -gt 1 ]; then
        local oldest_key_id=${key_ids[0]}
        gcloud iam service-accounts keys delete "$oldest_key_id" \
            --iam-account="$service_account" \
            --project="$PROJECT_ID" -q
        log_message "Deleted oldest key for $service_account: $oldest_key_id"
    else
        log_message "Only one key exists; no deletion necessary."
    fi

    # Clean up the local key file
    rm "$temp_key_file"
}

# Main loop to iterate over each service account
log_message "Starting key rotation for all service accounts"
for service_account in "${SERVICE_ACCOUNTS_ARRAY[@]}"; do
    rotate_key "$service_account" || log_message "Key rotation failed for $service_account"
done

log_message "Key rotation completed for all service accounts."
