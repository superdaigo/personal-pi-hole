# Personal Pi-hole

This is my work log creating a personal pi-hole(https://pi-hole.net) machine on the GCP for Android private DNS.

I expect to speed up my mobile internet by dropping unnecessary network traffic.

Also, it improves my personal internet security and privacy.

To me: Don't forget to change the private DNS settings on my phone.


Thank you @varunsridharan for the great article.

- https://blog.svarun.dev/configure-pi-hole-with-dns-over-tls-private-dns


## Requirement

- Domain name and access to the name server
- Google Cloud Platform

## Setup

### GCP Compute Engine

- Instance type: `e2-micro` (utilize free tire)
- OS: Ubuntu 22.04 LTS

See more details in [Create GCP Resources](create-gcp-resources.md)

SSH to the instance from the GCP web console.

### Release port 53 from systemd-resolved

Reference: https://www.linuxuprising.com/2020/07/ubuntu-how-to-free-up-port-53-used-by.html

Modify `/etc/systemd/resolved.conf`

``` bash
echo "# Lookup Google DNS
DNS=8.8.8.8
DNSStubListener=no" | \
  sudo tee -a /etc/systemd/resolved.conf
```

Confirm

```shell
$ grep -v  '^#' /etc/systemd/resolved.conf

[Resolve]
DNS=8.8.8.8
DNSStubListener=no
```

Restart `systemd-resolved`

```bash
sudo systemctl restart systemd-resolved
```


### Clone repository

Clone this repository under the `/opt` directory.

```bash
cd /opt
sudo git clone https://github.com/superdaigo/personal-pi-hole.git
```

Modify `__DOMAIN_NAME__` with your DNS server's name in the [docker-compose.yaml](docker-compose.yaml) file.

confirmation

``` bash
grep DOMAIN_NAME: personal-pi-hole/docker-compose.yaml
```

### Install docker

See the following url.
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository


### Certbot (Let's Encrypt)

Certbot is a probram to obtain/renew/delete a TLS certificate issued by Let's Encrypt.

- Let's Encrypt: https://letsencrypt.org
- Certbot: https://certbot.eff.org

Pull certbot image
- Certbot docker image: https://hub.docker.com/r/certbot/certbot/

``` bash
sudo docker pull certbot/certbot
```

Create a new TLS certificate (DNS authentication)

Replace `__EMAIL_ADDRESS__` and `__DOMAIN_NAME__` with proper values.

``` bash
cd /opt/personal-pi-hole

sudo docker run -it --rm \
  -v ./etc-letsencrypt/:/etc/letsencrypt/:rw \
  certbot/certbot \
  certonly \
  --manual \
  --agree-tos \
  --email __EMAIL_ADDRESS__ \
  --preferred-challenges dns \
  -d __DOMAIN_NAME__
```

Update your DNS record.

Start pi-hole

```bash
sudo docker compose up -d
```


## Pi-hole Web Admin Console

In this procedure, port 80 is not opened to the internet intentionally.

So, IAP tunnel is needed to connect to the Pi-hole web admin console.

```bash
gcloud compute start-iap-tunnel __INSTANCE_ID__ 80 \
  --local-host-port=localhost:10001 \
  --zone=__ZONE__ \
  --project __GCP_PROJECT_ID__
```

http://localhost:10001/admin/


### Reset admin password

A random Pi-hole admin password is generated by pihole docker image.
You can see the password in the console output. But you can reset the password using the following command.

```bash
sudo docker exec pihole sudo pihole -a -p __COMPLEX_PASSWORD__
```


## Certificate renewal

TODO: Check later
