# Cluster Software

At this point the cluster should be up and running and we can access it from our local machine. The rest of the guide will assume that you are running the commands from a windows machine.

## Longhorn

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

## Cert-Manager

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

## Ingress-Nginx

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