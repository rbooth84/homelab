# Technitium DNS Server

I have 2 dedicated VM's running [Technitium DNS Server](https://technitium.com/dns/) on 2 different Proxmox nodes. Each VM has 512mb ram and 5gb of storage.

## Prerequisites

```bash
sudo apt install curl certbot python3-certbot-dns-cloudflare -y
```

## Install

```bash
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

This is really all you need to install Technitium DNS Server directly on the host. You should now be able to access the DNS Server from the UI e.g. http://172.16.53.1:5380/

## Setup Certbot

1. Create Directories
    ```bash
    sudo mkdir /etc/letsencrypt/credentials/
    sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
    sudo mkdir /tmp/letsencrypt-acme-challenge
    ```

2. Create the `cloudflare.ini` file

    ```bash
    sudo nano /etc/letsencrypt/credentials/cloudflare.ini
    ```
    And paste these contents replacing the tokens with your values
    ```ini
    dns_cloudflare_email={cloudflare-email}
    dns_cloudflare_api_key={cloudflare-apiKey}
    ```
3. Create the `letsencrypt.ini` file
    ```bash
    sudo nano /etc/letsencrypt/credentials/cloudflare.ini
    ```
    And paste in these contents
    ```ini
    text = True
    non-interactive = True
    webroot-path = /tmp/letsencrypt-acme-challenge
    key-type = ecdsa
    elliptic-curve = secp384r1
    preferred-chain = ISRG Root X1
    ```

4. Run the following commands
    ```bash
    sudo chmod 644 /etc/letsencrypt/letsencrypt.ini
    sudo chmod 600 /etc/letsencrypt/credentials/cloudflare.ini
    ```

5. Create `convert_to_pkcs12.sh` to convert the keys to something the DNS server can use
    ```bash
    sudo nano /etc/letsencrypt/renewal-hooks/deploy/convert_to_pkcs12.sh
    ```
    And paste in these contents
    ```bash
    openssl pkcs12 -export \
        -in /etc/letsencrypt/live/technitium-dns/fullchain.pem \
        -inkey /etc/letsencrypt/live/technitium-dns/privkey.pem \
        -out /etc/letsencrypt/live/technitium-dns/technitium-dns.pfx \
        -passout pass: \
        -name "Technitium DNS Cert"
    ```

6. Run Certbot to create a certificate
    ```bash
    certbot certonly --config "/etc/letsencrypt/letsencrypt.ini" \
    --work-dir "/tmp/letsencrypt-lib" --logs-dir "/tmp/letsencrypt-log" \
    --cert-name "technitium-dns" --agree-tos \
    --email "{email}" \
    --domains "{domain}" \
    --authenticator dns-cloudflare \
    --dns-cloudflare-credentials "/etc/letsencrypt/credentials/cloudflare.ini" \
    --deploy-hook "/etc/letsencrypt/renewal-hooks/deploy/convert_to_pkcs12.sh"
    ```

7. Setup a cron job to automaticly renew the cert
    ```bash
    sudo crontab -e
    ```
    Now add this line at the bottom
    ```
    19 4 * * * /usr/bin/certbot renew --deploy-hook "/etc/letsencrypt/renewal-hooks/deploy/convert_to_pkcs12.sh"
    ```

## Configuration

When you first go into the Web UI it will ask you to set a admin password. Go ahead and set it, you can rename the admin usern under `Administration -> Users`

`Settings -> General`
Change the `DNS Server Domain` to a domain for the DNS server e.g. `dns1.mydomain.org` and Save Settings.

`Settings -> Web Service`

1. Set `Web Service HTTP Port` to Port `80`
2. HTTPS Options
    - Enable HTTPS
    - Enable HTTP to HTTPS Redirection
    - Enable Use A Self Signed TLS Certificate When TLS Certificate File Path Is Unspecified
3. Set `Web Service HTTPS Port` to port `443`
4. Set `TLS Certificate File Path` to `/etc/letsencrypt/live/technitium-dns/technitium-dns.pfx`
5. Save Settings

`Settings -> Forwarders` in the `Forwarders` section use the Quick Select and select `Cloudflare (DNS-over-UDP)` and Save Settings

### Install Split Horizon Support

1. Goto the Apps Tab
2. Click on the App Store button
3. Look for `Split Horizon` and then click the Install Button next to it
4. click the Config button next to the Split Horizon app
5. Add the following in the root object at the top
    ```json
    "networks": {
        "lan": [
            "172.16.0.0/16"
        ],
        "tailscale": [
            "100.64.0.0/10"
        ]
    },
    ```
    The lan network is my home network. The tailscale network is the network that tailscale creates IP address in, Keep it in there if using tailscale.
6. Save

## Adding the First Zone

Goto `Zones` then click `Add Zone`. Give the zone a name e.g. `mydomain.org` and keep the type as `Primary Zone (default)` then click the Add button.

## Split Horizon Record

1. Add a new record to the zone.
2. Name: `cluster-ingress.mydomain.org`
3. Type: `APP`
4. App Name: `Split Horizon`
5. Class Path: `SplitHorizon.SimpleAddress`
6. Record Data:
    ```json
    {
        "lan": [
            "172.16.8.11"
        ],
        "tailscale": [
            "100.74.112.113"
        ]
    }
    ```
