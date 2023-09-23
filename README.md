# Personal Pi-hole

This is my work log creating a personal pi-hole(https://pi-hole.net) machine on the GCP.

With Pi-hole, I expect web browsing on my personal phone to speed up by dropping unnecessary HTTP requests.

Also, it improves my personal internet security and privacy.

To me: Don't forget to change the private DNS settings on my phone.

## How to use this repository

Clone this repository under the `/opt` directory.

```shell
cd /opt
git clone https://github.com/superdaigo/personal-pi-hole.git
```

Start pi-hole

```shell
sudo docker compose up -d
```

Set Pi-hole admin site password

```shell
sudo docker exec pihole sudo pihole -a -p __COMPLEX_PASSWORD__
```

Connect to the Pi-hole web console through the GCP IAP tunnel.

```shell
gcloud compute start-iap-tunnel __INSTANCE_ID__ 80 \
  --local-host-port=localhost:10001 \
  --zone=__ZONE__ \
  --project __GCP_PROJECT_ID__
```

http://localhost:10001/admin/


## Setup

### GCP Compute Engine

Instance type: `e2-micro` (Utilize free tire)
OS: Ubuntu 22.04
Network tag: `dns-server`

Firewall rule:
Create the following
- Name: allow-dns (Incoming: 0.0.0.0/0, Port: 53, Protocol: UDP and TCP, Target tag: `dns-server`)
- Name: allow-iap-ssh (Incoming: 35.235.240.0/20, Port: 22, Target is all instance in the network)
- Name: allow-iap-http (Incoming: 35.235.240.0/20, Port: 80, Target is all instance in the network)
  
Delete default incoming rules allowing access from the internet.

SSH to the instance from the GCP web console.

### Release port 53 from systemd-resolved

Reference: https://www.linuxuprising.com/2020/07/ubuntu-how-to-free-up-port-53-used-by.html

Modify `/etc/systemd/resolved.conf`

```shell
$ grep -E '^(DNS|DNSStubListener)' /etc/systemd/resolved.conf
DNS=8.8.8.8
DNSStubListener=no
```

Restart `systemd-resolved`

```bash
sudo systemctl restart systemd-resolved
```
