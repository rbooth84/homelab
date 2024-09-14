# Kubernetes Cluster Setup

You will need atleast 1 server/vm to install Kubernetes on. In my home lab I have a 5 server proxmox cluster setup with 1 VM on each proxmox node that I installed my environment into it.

You will need the following:
- 1 or more Servers/VMs wuth Linux installed
- Each server should have a Static IP Address
- 1 IP to assign to kube-vip for the Control Plan
- IP Range to assign to MetalLB or kube-vip

> **_NOTE:_**  MetalLB is optional and you can just use kube-vip to handle load balancing for both the control plan and services. I chose to go with MetalLB because of some issues that I ran into with my homelab while using kube-vip. This guide will step you through setting up either one.

## Planning

Start by figuring out which IP you would like to assign to kube-vip for the control plan and a IP range that you would like to use for the Load Balancer, You will need atleast 2 in this range to follow my guide.

For my envionment I have 172.16.0.0/16 IP Range.

My Setup
- Server 1: k8s01 - 172.16.8.1
- Server 2: k8s02 - 172.16.8.2
- Server 3: k8s03 - 172.16.8.3
- Server 4: k8s04 - 172.16.8.4
- Server 5: k8s05 - 172.16.8.5
- kube-vip - 172.16.8.10
- IP Range for MetalLB - 172.16.8.11-172.16.8.254

## Prerequisitues

Install the following packages on each node in the cluster

```bash
sudo apt-get update
sudo apt-get install jq open-iscsi nfs-common curl -y
```

# K3s Install

Before we install kube-vip it requires containerd or docker to run their tool that creates a manifest for us. We will setup the first node in the cluster since it installs containerd then use that to run their tool.

## Initial Node

Run this command on the first server replacing `172.16.0.10` with your kube-vip ip address.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.28.12+k3s1" sh -s - server \
  --tls-san 172.16.0.10 \
  --cluster-init \
  --disable=servicelb,traefik \
  --write-kubeconfig-mode 644
```

After the command is ran you can run `kubectl get nodes` to check to see if it is up and running.

At this point we have a working kubernetes install without anything setup on it.

## Install Kube-VIP

1. Run the following command to setup the rbac for kube-vip
    ```bash
    kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
    ```

2. Setup the variables required for Generating the Kube-VIP Manifest. Change `172.16.0.10` to the IP you want to use for kube-vip virtual ip. It will be the same ip that you used as the `--tls-san`

    ```bash
    export VIP=172.16.0.10
    export INTERFACE=$(ip route | grep default | awk '{print $5}')
    KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
    alias kube-vip="sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
    ```

3. Create the Kube-VIP Manifest
    - if you would like to use kube-vip instead of MetalLB add `--services` to the command.

    ```bash
    kube-vip manifest daemonset \
        --interface $INTERFACE \
        --address $VIP \
        --inCluster \
        --taint \
        --controlplane \
        --arp \
        --leaderElection > kube-vip.yaml
    ```
    If using `--services` to use kube-vip instead of MetalLB run the following commands replacing the ip range wiith yours:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kube-vip/kube-vip-cloud-provider/main/manifest/kube-vip-cloud-controller.yaml
    kubectl create configmap -n kube-system kubevip --from-literal range-global=172.16.8.11-172.16.8.254
    ```

4. Apply the mainifest

    ```bash
    kubectl apply -f kube-vip.yaml
    ```

5. Verify that the Virtual IP is working

   - Run `kubectl get pods -n kube-system`
     - You should see a pod with a name that starts with `kube-vip-ds-` and it should have a running status
   - Open your browser to your kube-vip ip e.g. https://172.16.0.10:6443/. The browser will give you some security warnings about not being secure.

## Install MetalLB

If you chose to use Kube-VIP for services this can be skipped

1. Install the manifest
    ```powershell
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
    ```

2. Create a file called `metallb-config.yaml` and put in the IP Range you would like the  Load Balancer to use.

    ```yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: default-pool
      namespace: metallb-system
    spec:
      addresses:
      - 172.16.8.11-172.16.8.254
    ---
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: default
      namespace: metallb-system
    spec:
      ipAddressPools:
      - default-pool
    ```

3. Apply the changes

    ```powershell
    kubectl apply -f metallb-config.yaml
    ```

4. Verify that the MetalLB pods are ready

    ```powershell
    kubectl get pods -n metallb-system
    ```
    You should see a single controller pod and a speaker pod for each node.

## Patch local-storage

The way k3s works it automaticly applys the mainfests files located in `/var/lib/rancher/k3s/server/manifests/`. Since we will be setting up longhorn later and want that to be our default storage provider we should change the `is-default-class` setting in the `local-storage.yaml` file so that it does not change our setting later if you reboot or restart the k3s service on this node. We will need to apply this change to each node in the cluster as we install k3s.

If you run the following command you can see the storage classes on the cluster and see that local-storage is setup as the default.

```bash
kubectl get storageclass
```

Run this command to patch the file. This will change the setting to `false`

```bash
sudo sed -i 's/storageclass.kubernetes.io\/is-default-class: "true"/storageclass.kubernetes.io\/is-default-class: "false"/' "/var/lib/rancher/k3s/server/manifests/local-storage.yaml"
```

Now lets patch it in the cluster:

```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Run this command again and you will see it is no longer the default storage class

```bash
kubectl get storageclass
```

## Get the cluster token

Copy the token out of the `node-token` file so we can use it to setup additional nodes
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

The token looks like this

```
U10a82bcf64dc6e11d4f2605468c84873ebabcedda2ffddfdd0e91e8e3574e77788::server:74836d5735e88b03ee76b8be623bae7d
```

## Additional Nodes

Do the following on each node you would like to add to the cluster

1. Install K3s - Paste the Token and replace the `172.16.0.10` below with your kube-vip IP
    
    ```bash
    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.28.12+k3s1" sh -s - server \
        --token={Paste-Token-From-Prev-Step-Here} \
        --tls-san "172.16.0.10" \
        --server https://172.16.0.10:6443 \
        --disable=servicelb,traefik \
        --write-kubeconfig-mode 644
    ```

2. Patch the `local-storage.yaml`

    ```bash
    sudo sed -i 's/storageclass.kubernetes.io\/is-default-class: "true"/storageclass.kubernetes.io\/is-default-class: "false"/' "/var/lib/rancher/k3s/server/manifests/local-storage.yaml"
    ```

3. Patch the local-storage class running on the cluster.

    ```bash
    kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
    ```

4. Verify the node was added to the cluster

    ```bash
    kubectl get nodes
    ```