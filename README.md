# Homelab

This is a step by step guide on how my homelab is setup and configured.


## Hardware

- OPNsense Router
    - HP T620+
        - 256gb ssd
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
        - 64gb RAM
        - 500gb NVMe
- Development Kubernetes Cluster
    - 4x Raspberry Pi 4 8GB

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

## Remote Access

Remote Access is all connected using Tailscale.

## Kubernetes Cluster Setup Guides

1. [Kubernetes Cluster](Kubernetes-Install.md)
2. [Postgres HA Cluster using CloudNativePG](CloudNativePG.md)
3. [Single sign-on (SSO) with Authentik](Authentik.md)