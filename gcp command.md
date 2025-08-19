# How to filter traffic in gcp to allow all traffic from all instance
## Step-1: Add tag to the instance

```gcloud compute instances add-tags prom-a \
    --zone=us-central1-c \
    --tags=allow-all
```
## Step-2: Create firewall rule for all traffic with target as a tag
```
gcloud compute firewall-rules create allow-all-for-tag \
    --direction=INGRESS \
    --priority=1000 \
    --network=default \
    --action=ALLOW \
    --rules=all \
    --source-ranges=0.0.0.0/0 \
    --target-tags=allow-all
```
