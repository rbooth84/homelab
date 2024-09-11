# Exposing External Services through Kubernetes Ingress

In my homelab I have Gitea installed on my NAS but I also like to allow friends to connect to my Gitea instance so it needs to be exposed directly to the internet along with other services that I also have hosted in my Kubernetes Cluster.

Because ports 80 and 443 can only be exposed to a single service in my network from my internet providers IP address we need a reverse proxy to share it with multiple services hosted on different machines. Luckly that is excatly what a Ingress in Kubernetes is.

In this guide I will step you through exposing the Gitea instance to the Kubernetes cluster then creating an ingress for it which also includes lets encrypt certs from cert-manager.

## Creating the manifest

Create a file called `gitea-ingress.yaml`

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-gitea
subsets:
  - addresses:
      - ip: 172.16.5.8
    ports:
      - port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: external-gitea
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: external-gitea
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "400m"
spec:
  ingressClassName: nginx-external
  rules:
  - host: source.mydomain.org
    http:
      paths:
      - backend:
          service:
            name: external-gitea
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - source.mydomain.org
    secretName: gitea-tls
```

### Here is a break down of what is happenning

1. An Endpoint is created for that points to my NAS
    - `ip: 172.16.5.8`
        - This is the IP address of my NAS
    - `port: 8080`
        - This is the port that Gitea is exposed as on my NAS
2. ClusterIP Service is created in Kubernetes for use in the ingress
    - `port: 80`
        - The port that is exposed in kubernetes cluster
    - `targetPort: 8080`
        - The target port of my endpoint which is the port on the NAS for Gitea
3. Ingress is created to expose my service
    - `ingressClassName: nginx-external`
        - Tells Kubernetes to use my externaly exposed ingress-nginx instance. If **removed** the **default** ingress will be used instead.
    - `cert-manager.io/cluster-issuer: letsencrypt-prod`
        - Tells Cert-Manager to use my letsencrypt ClusterIssuer that I setup for requesting certificates.
    - `nginx.ingress.kubernetes.io/proxy-body-size`
        - Expanded the body size from the default to allow bigger files to be uploaded.


### Apply the manifest

```powershell
kubectl apply -f gitea-ingress.yaml
```

### Taking a closer Look

Run the following command

```Powershell
kubectl get endpoints,service,ingress external-gitea
```

You will see an output that looks like this

```
NAME                       ENDPOINTS         AGE
endpoints/external-gitea   172.16.5.8:8080   16d

NAME                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/external-gitea   ClusterIP   10.43.15.63   <none>        80/TCP    16d

NAME                                       CLASS            HOSTS                  ADDRESS        PORTS     AGE
ingress.networking.k8s.io/external-gitea   nginx-external   source.mydomain.org   172.16.8.12   80, 443   16d
```

This shows you Endpoint, Service and Ingress that was created when we applied the manifest.

You should now be able to access your application from https://source.mydomain.org