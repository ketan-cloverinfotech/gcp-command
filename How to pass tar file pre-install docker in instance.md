## Step 1: Create bucket
#### In GCP search GCS and create specific bucket name
eg.ketan-bucket-2026
#### Now open shell in GCP and create following variable
```
PROJECT_ID="$(gcloud config get-value project)"
ZONE="us-central1-a"          # change if you want
VM_NAME="rhel-docker-1"       # change if you want
MACHINE_TYPE="e2-standard-2"

BUCKET="ketan-bucket-2026"
TAR_OBJECT="alfresco.tar"
TAR_GCS_URI="gs://$BUCKET/$TAR_OBJECT"

SA_NAME="vm-bootstrap-sa"
SA_EMAIL="$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"
```
## Step 2) Create a service account and give it read access to the bucket
```
gcloud iam service-accounts create "$SA_NAME" \
  --display-name="VM Bootstrap SA" || true

gcloud storage buckets add-iam-policy-binding "gs://$BUCKET" \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/storage.objectViewer"
```
## Step 3) Create the startup script file
```
cat > startup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

LOG=/var/log/startup-script.log
exec > >(tee -a "$LOG") 2>&1

echo "=== Startup script started at $(date) ==="

# ---- CONFIG ----
TAR_GCS_URI="__TAR_GCS_URI__"
DEST_DIR="/opt/alfresco_bootstrap"
DONE_FLAG="/var/lib/startup-script.done"
# --------------

# Prevent re-running on every reboot (GCP startup script runs each boot)
if [[ -f "$DONE_FLAG" ]]; then
  echo "Startup already completed ранее; exiting."
  exit 0
fi

mkdir -p "$DEST_DIR"

echo "Installing tools..."
dnf -y install nano tree tar gzip curl ca-certificates dnf-plugins-core

echo "Installing Docker..."
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

echo "Ensuring gsutil exists..."
if ! command -v gsutil >/dev/null 2>&1; then
  tee /etc/yum.repos.d/google-cloud-sdk.repo >/dev/null <<'REPO'
[google-cloud-cli]
name=Google Cloud CLI
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el9-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
REPO
  dnf -y install google-cloud-cli
fi

echo "Downloading tar from: $TAR_GCS_URI"
gsutil cp "$TAR_GCS_URI" "$DEST_DIR/"

TAR_NAME="$(basename "$TAR_GCS_URI")"
mkdir -p "$DEST_DIR/extracted"

echo "Extracting $TAR_NAME ..."
tar -xvf "$DEST_DIR/$TAR_NAME" -C "$DEST_DIR/extracted"

touch "$DONE_FLAG"
echo "=== Startup script finished at $(date) ==="
EOF

```
## Step 4 Create the RHEL 9 VM with the startup script
```
gcloud compute instances create "$VM_NAME" \
  --zone="$ZONE" \
  --machine-type="$MACHINE_TYPE" \
  --image-family="rhel-9" \
  --image-project="rhel-cloud" \
  --service-account="$SA_EMAIL" \
  --scopes="https://www.googleapis.com/auth/cloud-platform" \
  --metadata-from-file startup-script=./startup.sh
```
## Step 5: After vm is created verify the logs 
```
gcloud compute ssh "$VM_NAME" --zone="$ZONE" --command "sudo docker --version; sudo ls -lah /opt/alfresco_bootstrap; sudo tail -n 200 /var/log/startup-script.log"

```
