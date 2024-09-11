# Kubernetes Cluster Setup
This will guide you on how to setup Kubernetes with the following features using K3s.

- [Rancher](https://www.rancher.com/) as a UI for the cluster.
- [Longhorn](https://longhorn.io/) for storage
- [cert-manager](https://cert-manager.io/) for Let's Encrypt Certs with Cloudflare DNS Validation
- [Ingress-NGINX](https://github.com/kubernetes/ingress-nginx) as a reverse proxy into the cluster
- [kube-vip](https://kube-vip.io/) as a LoadBalancer for the Control Plane
- [MetalLB](https://metallb.universe.tf/) as a LoadBalancer for Services
- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator) for Remote Access

## Getting Started
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

# Setting up Local Cluster Access

Now that the cluster is up and running lets install kubectl localy and install helm.

This guide will assume [Chocolatey](https://chocolatey.org/install) is installed in Windows.

## Install Kubectl

```powershell
choco install kubernetes-cli
```

## Kubectl Config

Grab the kube config file from any node:

```bash
cat /etc/rancher/k3s/k3s.yaml
```

Copy the contents of the file and place them in your user folder. Inside your user folder, create a new folder named `.kube`. Then, inside the `.kube` folder, create a file named `config`. The final file path will be `C:\Users\myuser\.kube\config`.

Verify that kubectl is working by running this command:

```powershell
kubectl get nodes
```

## Install Helm

```powershell
choco install kubernetes-helm
```

Verify that helm is working:
```powershell
helm ls
```

This command list installed releases on your cluster. The list will be empty since we have not installed any helm packages onto the cluster.

# Cluster Software

At this point the cluster should be up and running and we can access it from our local machine. The rest of the guide will assume that you are running the commands from a windows machine.

## Installing Longhorn

1. Setup the helm repo

    ```powershell
    helm repo add longhorn https://charts.longhorn.io 
    helm repo update
    ```
2. Install the Helm Chart

    ```powershell
    helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
    ```

3. Verify the Longhorn Pods are up

    ```powershell
    kubectl get pods -n longhorn-system
    ```

    After a few seconds you should see all the pods for longhorn become ready.

### Accessing the longhorn UI

Since we don't currently have ingress-nginx installed on the cluster we will setup a NodePort Service so we can access the UI so we can take a look.

Create a file called `longhorn-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-nodeport
  namespace: longhorn-system
spec:
  type: NodePort
  selector:
    app: longhorn-ui
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
    nodePort: 30080
```

Apply the manifest

```powershell
kubectl apply -f longhorn-service.yaml
```

This will create a NodePort Service that that you can access from any node the cluster. Access the UI from using the initial server e.g. http://172.16.8.1:30080/. If everything is working you should see the dasboard.

Remove the service once you are done.

```powershell
kubectl delete -f longhorn-service.yaml
```

Longhorn does not have any built in security. After you complete the kubernetes setup follow [my guide](Authentik.md) on setting up [Authentik](https://goauthentik.io/) which also includes setting up ingress for Longhorn with SSO.

## Install Cert-Manager

In my environment I have a .org domain with Cloudflare DNS Validation configured so that I dont have to expose anything from my internal environment to get valid ssl certs.

1. Setup the helm repo

    ```powershell
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    ```

2. Install the helm chart

    ```powershell
    helm install cert-manager jetstack/cert-manager `
      --namespace cert-manager `
      --create-namespace `
      --version v1.12.13 `
      --set installCRDs=true `
      --set podDnsPolicy=None `
      --set podDnsConfig.nameservers[0]=1.1.1.1 `
      --set podDnsConfig.nameservers[1]=1.0.0.1 `
      --set webhook.securePort=10260
    ```

3. Create a file called `cluster-issuer.yaml` and put in your Cloudflare Email and [Cloudflare API Token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        email: <Your-Email-Here>
        server: https://acme-v02.api.letsencrypt.org/directory
        privateKeySecretRef:
          name: le-issuer-acct-key
        solvers:
        - dns01:
            cloudflare:
              email: <Your-Cloudflare-Email-Here>
              apiTokenSecretRef:
                name: cloudflare-api-token-secret
                key: api-token
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api-token-secret
      namespace: cert-manager
    type: Opaque
    stringData:
      api-token: <Your-Cloudflare-API-Token-Here>
    ```

4. Now apply the manifest

    ```powershell
    kubectl apply -f cluster-issuer.yaml
    ```

## Install Ingress-Nginx

In my environment, I like to have two separate installations of ingress-nginx, one for internal use and another for external, which I expose over NAT through my home router. This setup helps prevent accidentally exposing services to the internet. The internal ingress is configured as the default controller.

1. Setup the helm repo

    ```powershell
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    ```

2. Install the required CRDs

    ```powershell
    kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/v3.6.2/deploy/crds.yaml
    ```

3. Create a file called `nginx-internal-values.yaml`. Set the loadBalancerIP to the first IP in your load balancer range e.g. `172.16.8.11`.

    ```yaml
    controller:
      ingressClassByName: true
      allowSnippetAnnotations: true
      ingressClassResource:
        name: nginx-internal
        enabled: true
        default: true
        controllerValue: k8s.io/ingress-ngnix-internal
      service:
        loadBalancerIP: 172.16.8.11
    ```
    Notice that `default` is set to `true`

4. Install the helm chart

    ```powershell
    helm install ingress-nginx-internal ingress-nginx/ingress-nginx -f nginx-internal-values.yaml --namespace nginx-internal --create-namespace
    ```

    If you navigate to http://172.16.8.11/ in the browser you should see a 404 page with nginx in the footer.

5. Create a file called `nginx-external-values.yaml`. Set the loadBalancerIP to the 2nd in your load balancer range e.g. `172.16.8.12`

    ```yaml
    controller:
      ingressClassByName: true
      allowSnippetAnnotations: true
      ingressClassResource:
        name: nginx-external
        enabled: true
        default: false
        controllerValue: k8s.io/ingress-ngnix-external
      service:
        loadBalancerIP: 172.16.8.12
    ```

    Notice that `default` is set to `false`

6. Install the helm chart

    ```powershell
    helm install ingress-nginx-external ingress-nginx/ingress-nginx -f nginx-external-values.yaml --namespace nginx-external --create-namespace
    ```

    Navigate to http://172.16.8.12/ to verify that the nginx is up and running.

## Install Rancher

1. Create a DNS entry for your domain and point it to the internal ingress-nginx ip address or [add a entry into your host file](https://www.liquidweb.com/blog/edit-hosts-file-macos-windows-linux/).

2. Install the helm repo

    ```powershell
    helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
    helm repo update
    ```

3. Install Rancher. Make sure to change the hostname below for the domain that you will be using to access rancher.

    ```powershell
    helm install rancher rancher-latest/rancher `
      --namespace cattle-system `
      --set hostname=rancher.mydomain.org `
      --set bootstrapPassword=admin `
      --set ingress.tls.source=secret `
      --set ingress.extraAnnotations.'certmanager\.k8s\.io/cluster-issuer'=letsencrypt-prod
    ```

4. Create a file called `rancher-ingress.yaml` with the contents below. Make sure to swap out the domain with your own.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: rancher-ingress
      namespace: cattle-system
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
    spec:
      rules:
      - host: rancher.mydomain.org
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rancher
                port:
                  number: 80
      tls:
      - hosts:
        - rancher.mydomain.org
        secretName: rancher-tls
    ```

5. Apply the manifest

    ```powershell
    kubectl apply -f rancher-ingress.yaml
    ```

    It take about 75 seconds for cert-manager to get the certificate. You can view the status of the cert with this command

    ```powershell
    kubectl get certs -n cattle-system
    ```

6. Naviage to the install with the domain you setup e.g. https://rancher.mydomain.org/. When asked the bootstrap password its `admin`. This was set in the helm install command.

## Tailscale Operator

In my environment I have [Technitium DNS Server](https://technitium.com/dns/) installed and configured with the Split Horizon app and is exposed on the Tailscale network. This allows me to direct to the correct IP based on if its coming from Tailscale or my LAN. In Tailscale I configured DNS to forward DNS Quries for mydomain.org to the Tailscale IP of my DNS server.

The TailScale operator Docs can be found [here](https://tailscale.com/kb/1236/kubernetes-operator)


1. Login to Tailscale
2. Setup the Access Controls.

    - Add the following [here](https://login.tailscale.com/admin/acls/file)
    ```json
    "tagOwners": {
      "tag:k8s-operator": [],
      "tag:k8s":          ["tag:k8s-operator"],
    },
    ```

3. Create a OAuth Client [Here](https://login.tailscale.com/admin/settings/oauth)

    1. Description: k8s
    2. Under All give it Read/Write
    3. Click Generate client
    4. Copy the Client ID and Client secret. You will not see this again.

4. Install Helm Repo

    ```powershell
    helm repo add tailscale https://pkgs.tailscale.com/helmcharts
    helm repo update
    ```

5. Install Helm Chart. Replace with client id and secret from step 3.3

    ```powershell
    helm upgrade `
      --install `
      tailscale-operator `
      tailscale/tailscale-operator `
      --namespace=tailscale `
      --create-namespace `
      --set-string oauth.clientId=<OAauth client ID> `
      --set-string oauth.clientSecret=<OAuth client secret> `
      --wait
    ```

6. Verify that the operator pod is running

    ```powershell
    kubectl get pods -n tailscale
    ```

7. Lets setup an service to connect it to our internal ingress-nginx controller. Create a file called `nginx-internal-tailscale-svc.yaml`:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: ingress-tailscale
      namespace: nginx-internal
    spec:
      type: LoadBalancer
      loadBalancerClass: tailscale
      ports:
      - appProtocol: http
        name: http
        port: 80
        protocol: TCP
        targetPort: http
      - appProtocol: https
        name: https
        port: 443
        protocol: TCP
        targetPort: https
      selector:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx-internal
        app.kubernetes.io/name: ingress-nginx
    ```

8. Apply the manifest

    ```powershell
    kubectl apply -f nginx-internal-tailscale-svc.yaml
    ```

9. Verify that everything is working

    The tailscale operator will create a device on the tailscale network called `nginx-internal-ingress-tailscale`. For me it has an IP of `100.76.113.121`. When I navigate to http://100.76.113.121/ while connected to the tailscale netowrk I can see the nginx 404 page.

    Tailscale will name the device like this {namespace}-{service-name}