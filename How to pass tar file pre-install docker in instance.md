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
## Step 6: When you want to create multiple instances
```
# Set once
PROJECT_ID="$(gcloud config get-value project)"
ZONE="us-central1-a"
MACHINE_TYPE="e2-standard-2"

# Bucket/tar
BUCKET="ketan-bucket-2026"
TAR_OBJECT="alfresco.tar"
TAR_GCS_URI="gs://$BUCKET/$TAR_OBJECT"

# Service Account (same as before)
SA_NAME="vm-bootstrap-sa"
SA_EMAIL="$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"

# (Optional) ensure SA exists + has bucket read
gcloud iam service-accounts create "$SA_NAME" --display-name="VM Bootstrap SA" 2>/dev/null || true
gcloud storage buckets add-iam-policy-binding "gs://$BUCKET" \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/storage.objectViewer" >/dev/null

# Create startup script file (same logic, with your tar)
cat > startup.sh <<'EOF'
#!/bin/bash
set -euo pipefail
LOG=/var/log/startup-script.log
exec > >(tee -a "$LOG") 2>&1

echo "=== Startup script started at $(date) ==="
TAR_GCS_URI="__TAR_GCS_URI__"
DEST_DIR="/opt/alfresco_bootstrap"
DONE_FLAG="/var/lib/startup-script.done"

if [[ -f "$DONE_FLAG" ]]; then
  echo "Startup already completed; exiting."
  exit 0
fi

mkdir -p "$DEST_DIR"

dnf -y install nano tree tar gzip curl ca-certificates dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

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

gsutil cp "$TAR_GCS_URI" "$DEST_DIR/"
TAR_NAME="$(basename "$TAR_GCS_URI")"
mkdir -p "$DEST_DIR/extracted"
tar -xvf "$DEST_DIR/$TAR_NAME" -C "$DEST_DIR/extracted"

touch "$DONE_FLAG"
echo "=== Startup script finished at $(date) ==="
EOF

sed -i "s|__TAR_GCS_URI__|$TAR_GCS_URI|g" startup.sh
chmod +x startup.sh

# Loop create: dc1 dc2 dr1 dr2
for VM_NAME in dc1 dc2 dr1 dr2; do
  echo "Creating $VM_NAME in $ZONE ..."
  gcloud compute instances create "$VM_NAME" \
    --zone="$ZONE" \
    --machine-type="$MACHINE_TYPE" \
    --image-family="rhel-9" \
    --image-project="rhel-cloud" \
    --service-account="$SA_EMAIL" \
    --scopes="https://www.googleapis.com/auth/cloud-platform" \
    --metadata-from-file startup-script=./startup.sh
done

```
### Step 7 When You want to add your ssh key to server
```
#!/bin/bash
set -euo pipefail

LOG=/var/log/startup-script.log
exec > >(tee -a "$LOG") 2>&1

echo "=== Startup script started at $(date) ==="

KEY_URI="gs://ketan-bucket-2026/rootkey.pub"
TAR_GCS_URI="gs://ketan-bucket-2026/alfresco.tar"
DEST_DIR="/opt/alfresco_bootstrap"
DONE_FLAG="/var/lib/startup-script.done"

mkdir -p /root/.ssh
chmod 700 /root/.ssh
touch /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

# Ensure gsutil exists
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

# Key fetch should NOT block rest of script
TMP_KEY="/tmp/rootkey.pub"
if gsutil ls "$KEY_URI" >/dev/null 2>&1; then
  gsutil cat "$KEY_URI" > "$TMP_KEY"
  grep -qxF "$(cat "$TMP_KEY")" /root/.ssh/authorized_keys || cat "$TMP_KEY" >> /root/.ssh/authorized_keys
  restorecon -RFv /root/.ssh || true
  echo "Root SSH key added."
else
  echo "WARNING: root key not found at $KEY_URI, skipping root authorized_keys step."
fi

# OPTIONAL: allow root SSH with keys only (comment if you don't want root SSH)
echo "PermitRootLogin prohibit-password" > /etc/ssh/sshd_config.d/99-rootlogin.conf
systemctl restart sshd || true

# Run heavy install only once
if [[ -f "$DONE_FLAG" ]]; then
  echo "Startup already completed; skipping install/copy steps."
  exit 0
fi

mkdir -p "$DEST_DIR"

dnf -y install nano tree tar gzip curl ca-certificates dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

gsutil cp "$TAR_GCS_URI" "$DEST_DIR/"
TAR_NAME="$(basename "$TAR_GCS_URI")"
mkdir -p "$DEST_DIR/extracted"
tar -xvf "$DEST_DIR/$TAR_NAME" -C "$DEST_DIR/extracted"

touch "$DONE_FLAG"
echo "=== Startup script finished at $(date) ==="
```
```
ZONE="us-central1-a"
MACHINE_TYPE="e2-standard-2"
PROJECT_ID="$(gcloud config get-value project)"
SA_EMAIL="vm-bootstrap-sa@$PROJECT_ID.iam.gserviceaccount.com"

for VM in dc1 dc2 dr1 dr2; do
  echo "Creating $VM..."
  gcloud compute instances create "$VM" \
    --zone="$ZONE" \
    --machine-type="$MACHINE_TYPE" \
    --image-family="rhel-9" \
    --image-project="rhel-cloud" \
    --service-account="$SA_EMAIL" \
    --scopes="https://www.googleapis.com/auth/cloud-platform" \
    --metadata-from-file startup-script=./startup.sh
done
```
## Verify logs of One VMs
```
gcloud compute ssh dc1 --zone us-central1-a --command \
"sudo tail -n 80 /var/log/startup-script.log; sudo tail -n 3 /root/.ssh/authorized_keys; sudo docker --version; sudo ls -lah /opt/alfresco_bootstrap"
```
