## Step 1: login with gcloud auth
```
gcloud auth login
```
## Step 2: Install he GKE gcloud auth plugin
```
# 1) Prereqs
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates gnupg curl

# 2) Add Google Cloud SDK APT repo + key
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
  | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

# 3) Install the plugin
sudo apt-get update
sudo apt-get install -y google-cloud-sdk-gke-gcloud-auth-plugin

# 4) Make sure kubectl can find it
export PATH=/usr/lib/google-cloud-sdk/bin:$PATH
export USE_GKE_GCLOUD_AUTH_PLUGIN=True

# 5) Sanity-check
which gke-gcloud-auth-plugin
gke-gcloud-auth-plugin --version

```
## Step 3: Install kubectl 
```
curl -LO https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl
chmod +x kubectl 
sudo cp kubectl /usr/local/bin 
```
