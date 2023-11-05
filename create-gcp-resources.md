# Create GCP resources

Create following resources.

- VPC x 1
- Subnet x 1
- Instance x 1
- Firewall rule x 4

## Environment variables

``` bash
PROJECT_NAME=pi-hole-12345  # Use your project name
NETWORK_NAME=pi-hole
REGION=us-west1
SUBNET_NAME=${NETWORK_NAME}-${REGION}

gcloud config set project "$PROJECT_NAME"
gcloud config set compute/zone "${REGION}-b"
```

## Create network

```bash
gcloud compute networks create "$NETWORK_NAME" \
  --subnet-mode=custom \
  --mtu=1460 \
  --enable-ula-internal-ipv6 \
  --bgp-routing-mode=regional

gcloud compute networks subnets create "$SUBNET_NAME" \
  --range=10.0.0.0/24 \
  --stack-type=IPV4_IPV6 \
  --ipv6-access-type=EXTERNAL \
  --network="$NETWORK_NAME" \
  --region="$REGION"
```

## Create Firewall rules

Create firewall rules specifying a target tag `pi-hole`.

- Name: allow-dns-over-tls (Incoming: 0.0.0.0/0, Port: 853, Protocol: TCP, Target tag: `pi-hole`)
- Name: allow-dns-over-tls-ipv6 (Incoming: ::/0, Port: 853, Protocol: TCP, Target tag: `pi-hole`)
- Name: allow-iap-ssh (Incoming: 35.235.240.0/20, Port: 22, Protocol: TCP, Target tag: `pi-hole`)
- Name: allow-iap-http (Incoming: 35.235.240.0/20, Port: 80, Protocol: TCP, Target tag: `pi-hole`)

### Allow DNS over TLS (IPv4 and IPv6)

``` bash
gcloud compute \
  firewall-rules create "${NETWORK_NAME}-allow-dns-over-tls" \
  --direction=INGRESS \
  --priority=1000 \
  --network="$NETWORK_NAME" \
  --action=ALLOW \
  --rules=tcp:853 \
  --source-ranges="0.0.0.0/0" \
  --target-tags=pi-hole

gcloud compute \
  firewall-rules create "${NETWORK_NAME}-allow-dns-over-tls-ipv6" \
  --direction=INGRESS \
  --priority=1000 \
  --network="$NETWORK_NAME" \
  --action=ALLOW \
  --rules=tcp:853 \
  --source-ranges="::/0" \
  --target-tags=pi-hole
```

### Allow SSH from IAP

``` bash
gcloud compute \
  firewall-rules create "${NETWORK_NAME}-allow-ssh-from-iap" \
  --direction=INGRESS \
  --priority=1000 \
  --network="$NETWORK_NAME" \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=pi-hole
```

### Allow HTTP from IAP

``` bash
gcloud compute \
  firewall-rules create "${NETWORK_NAME}-allow-http-from-iap" \
  --direction=INGRESS \
  --priority=1000 \
  --network="$NETWORK_NAME" \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=pi-hole
```

## Create instance

Instance Name: pi-hole-12345678
OS: Ubuntu 22.04 LTS
Zone: us-west1-b

``` bash
INSTANCE_NAME=pi-hole-$(head -c1024 /dev/urandom | sha512sum | head -c8)
PROJECT_NUMBER=$( \
  gcloud projects describe ${PROJECT_NAME} \
  --format 'value(projectNumber)'\
)
IMAGE_NAME=$(\
  gcloud compute images \
    describe-from-family ubuntu-2204-lts \
    --project ubuntu-os-cloud \
    --format "value(name)" \
)
ZONE="${REGION}-b"

echo "Going to create an instance...
Project Name (Number): $PROJECT_NAME ($PROJECT_NUMBER)
Base Image Name: $IMAGE_NAME
Instance Name: $INSTANCE_NAME
Region: $REGION
Zone: $ZONE
"

gcloud compute instances create "$INSTANCE_NAME" \
  --zone="$ZONE" \
  --machine-type=e2-micro \
  --network-interface=network-tier=PREMIUM,stack-type=IPV4_IPV6,subnet="$SUBNET_NAME" \
  --metadata=enable-osconfig=TRUE \
  --maintenance-policy=MIGRATE \
  --provisioning-model=STANDARD \
  --service-account="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --scopes="https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append" \
  --tags=pi-hole \
  --create-disk="auto-delete=yes,boot=yes,device-name=${INSTANCE_NAME},image=projects/ubuntu-os-cloud/global/images/${IMAGE_NAME},mode=rw,size=30,type=projects/${PROJECT_NAME}/zones/${ZONE}/diskTypes/pd-standard" \
  --no-shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring \
  --labels=goog-ops-agent-policy=v2-x86-template-1-1-0,goog-ec-src=vm_add-gcloud \
  --reservation-affinity=any
```

## Set DNS records

Create A and/or AAAA records of your private DNS server in your DNS server.

Show global IP addresses for DNS record registration.

``` bash
gcloud compute instances describe "$INSTANCE_NAME" \
  --format "text(
    networkInterfaces[0].accessConfigs[0].natIP,
    networkInterfaces[0].ipv6AccessConfigs[0].externalIpv6
  )"
```


## Delete

Delete all resources.

``` bash
gcloud compute instances list --format 'value(name)' | \
  xargs gcloud compute instances delete --quiet
  
gcloud compute firewall-rules delete "${NETWORK_NAME}-allow-dns-over-tls" \
  --project="$PROJECT_NAME" \
  --quiet

gcloud compute firewall-rules delete "${NETWORK_NAME}-allow-ssh-from-iap" \
  --project="$PROJECT_NAME" \
  --quiet

gcloud compute firewall-rules delete "${NETWORK_NAME}-allow-http-from-iap" \
  --project="$PROJECT_NAME" \
  --quiet

gcloud compute networks subnets delete "$SUBNET_NAME" \
  --region "$REGION" \
  --quiet

gcloud compute networks delete "$NETWORK_NAME" \
  --quiet
```
