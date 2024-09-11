# Homelab

This guide provides a detailed, step-by-step walkthrough of how I’ve set up and configured my homelab. From hardware selection to software deployment and access management, you’ll get an inside look at how everything fits together. Whether you're looking to replicate my setup or gain insights for your own, this guide covers the essentials of running a robust and flexible homelab environment.


## Hardware

- OPNsense Router
    - HP T620+
        - 12gb RAM
        - 256gb SSD
        - 4 port Intel NIC Card
- Siwtch
    - Zyxel 24-Port Gigabit Switch
- NAS
    - Synology 1821+
        - 64gb RAM
        - 2x - 1TB SSD in RAID1
        - 6x - 8TB HDD in RAID6
        - 2x - 500GB NVMe Cache Drives
- Proxmox Cluster
    - 5x Lenovo ThinkCentre M910Q
        - i7-7700T Processor
        - 64GB RAM
        - 500GB NVMe
- Development Kubernetes Cluster
    - 4x Raspberry Pi 4 8GB
- Remote Backups - Hosted at Parents house
    - Synology 220+
        - 16gb RAM
        - 2x TB in RAID1

## Software

- OPNSense
- On the NAS
    - Gitea
    - HashiCorp Vault
    - Watchtower
    - MinIO
    - Media Server Stack
    - Docker CloudFlare DDNS
- Proxmox
    - Home Assistant
    - Homebridge
    - Technitium DNS Server
        - Primary DNS Server
        - Secondary DNS Server
    - Active Directory
    - IIS Web Server
    - Azure DevOps
    - SQL Server 2022
    - Gitea Actions Runner
    - Windows Remote Desktop VM
    - Kubernetes Cluster
        - Longhorn
        - Rancher
        - Authentik
        - Homarr
        - Bookstack
        - Vaultwarden
        - Paperless-NGX
        - IT Tools
        - MariaDB with Mariadb-operator
        - Postgres with CloudNativePG
        - ArgoCD
        - Artifactory

## Access

### Public Access

Some services I like to expose directly to the internet. This is this is achieved by having a dedicated external ingress controller setup in kubernetes and exposing it to the internet using NAT in the router.

### Remote Access

Remote Access is achieved using Tailscale.

In Kubernetes I setup the Tailscale Operator and connected it to my internal ingress controller. Ingress is setup for all of my services inside and outside of my cluster to handle the remote connections.

For backups I have Tailscale installed on both Synology devices to allow secure connections to handle Hyper Backup and S3 Replication in MinIO.

## Kubernetes Cluster Setup Guides

1. [Kubernetes Cluster Setup](Kubernetes-Cluster-Setup.md)
2. [Postgres HA Cluster using CloudNativePG](CloudNativePG.md)
3. [Single sign-on (SSO) with Authentik](Authentik.md)
4. [Exposing External Services through Ingress](External-Ingress.md)