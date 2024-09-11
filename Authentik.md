# Authentik

This will guide on on setting up [Authentik](https://goauthentik.io/) in the Kubernetes cluster to enable Single sign-on (SSO) in your environment.

## Install

I have a free [mailgun](https://www.mailgun.com/) account that I use for sending emails. When setting up change the settings to match your smtp provider settings.

1. Setup the helm repo
    ```powershell
    helm repo add authentik https://charts.goauthentik.io
    helm repo update
    ```

2. Setup a database and user in the `postgres-cluster` created with [CloudNativePG](CloudNativePG.md)

    - Use [this](https://www.enterprisedb.com/postgres-tutorials/how-create-postgresql-database-and-users-using-psql-and-pgadmin) guide if you need instructions
    - Database: `authentik`
    - User: `authentik`

3. Create a namespace

    ```powershell
    kubectl create namespace authentik
    ```

4. Create a file called `authentik-secrets.yaml` replacing the values with your own.
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: authentik-secrets
      namespace: authentik
    type: Opaque
    stringData:
      email_user: <postmaster@mydomain.org>
      email_pass: <my-mailgun-password>
      pg_pass: <pgPassword>
      secret_key: <secret-key>
    ```

    Run `openssl rand 60 | base64 -w 0` from any linux machine and past the value into the `secret_key` field.

5. Apply the secrets to the cluster

    ```powershell
    kubectl apply -f authentik-secrets.yaml
    ```

6. Create a file called `authentik-values.yaml`. Make sure to change the email settings and domain to match your own.
    ```yaml
    global:
        env:
        - name: AUTHENTIK_SECRET_KEY
            valueFrom:
                secretKeyRef:
                    name: authentik-secrets
                    key: secret_key
        - name: AUTHENTIK_POSTGRESQL__PASSWORD
            valueFrom:
                secretKeyRef:
                    name: authentik-secrets
                    key: pg_pass
        - name: AUTHENTIK_EMAIL__USERNAME
            valueFrom:
                secretKeyRef:
                    name: authentik-secrets
                    key: email_user
        - name: AUTHENTIK_EMAIL__PASSWORD
            valueFrom:
                secretKeyRef:
                    name: authentik-secrets
                    key: email_pass
    authentik:
        postgresql:
            host: postgres-cluster-rw.default.svc.cluster.local
        email:
            host: "smtp.mailgun.org"
            port: 465
            use_tls: false
            use_ssl: true
            timeout: 10
            from: "authentik@mydomain.org"
    server:
        ingress:
            enabled: true
            hosts:
            - auth.mydomain.org
            annotations:
                cert-manager.io/cluster-issuer: letsencrypt-prod
            ingressClassName: nginx-external
            tls:
            - hosts:
                - auth.mydomain.org
            secretName: authentik-server-tls
    postgresql:
        enabled: false

    redis:
        enabled: true
        master:
            persistence:
                size: 1Gi
    ```
7. Install the helm chart
    ```powershell
    helm upgrade --install authentik authentik/authentik -f authentik-values.yaml --create-namespace --namespace authentik
    ```
8. Verify that the pods are all running

    ```powershell
    kubectl get pods -n authentik
    ```

The first time it starts up it might take a few minutes as it gets everything up and running.

Once the application is up you can access the setup screen at https://auth.mydomain.org/if/flow/initial-setup/ to finish setup.

## Longhorn Ingress Secured with Authentik

Now that we have a working install of Authentik lets setup ingress for longhorn and secure it using Authentik.

### Autheink Setup

1. Go into the Admin Area in Authentik
2. Goto Application -> Applicatiions
3. On the top click the `Create With Wizard` button
4. Application Details
    - Name: `Longhorn`
    - Slug: `longhorn`
    - UI Settings -> Launch URL: https://longhorn.mydomain.org
    - Click `Next`
5. Provider Type: Select `Forward Auth (Single Application)` then click `Next`
6. Provider Configuration
    - Name: `Provider for Longhorn`
    - Authentication flow: `default-authentication-flow (Welcome to authentik!)`
    - Authorization flow: `default-provider-authorization-implicit-consent (Authorize Application)`
    - External Host: `https://longhorn.mydomain.org`
    - Click `Submit`
7. Now goto Application -> Outposts
8. Click the Edit button next to `authentik Embedded Outpost`
9. Under `Available Applications` select `Longhorn` and move it into the `Selected Applications`
10. Save

Authentik is now ready to accept logins from longhorn.

### Longhorn Ingress Setup

Due to the way namespaces work in Kubernetes and the Forward Auth works with Authentik we need to create 2 different ingresses in Kubernetes. This is because a ingress in one namespace cannot connect to a service in another namespace.

1. Create a file called `longhorn-ingress.yaml`. Make sure to change `mydomain.org` to your domain name. It is in 5 different places.
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    name: longhorn-frontend--https
    namespace: longhorn-system
    annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        nginx.ingress.kubernetes.io/auth-url: |-
            http://authentik-server.authentik.svc.cluster.local/outpost.goauthentik.io/auth/nginx
        nginx.ingress.kubernetes.io/auth-signin: |-
            https://longhorn.mydomain.org/outpost.goauthentik.io/start?rd=$escaped_request_uri
        nginx.ingress.kubernetes.io/auth-response-headers: |-
            Set-Cookie,X-authentik-username,X-authentik-groups,X-authentik-email,X-authentik-name,X-authentik-uid
        nginx.ingress.kubernetes.io/auth-snippet: |
            proxy_set_header X-Forwarded-Host $http_host;
    spec:
    rules:
    - host: longhorn.mydomain.org
        http:
        paths:
        - path: /
            pathType: Prefix
            backend:
            service:
                name: longhorn-frontend
                port:
                number: 80
    tls:
    - hosts:
        - longhorn.mydomain.org
        secretName: longhorn-frontend-tls
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    name: longhorn-auth
    namespace: authentik
    spec:
    rules:
    - host: longhorn.mydomain.org
        http:
        paths:
        - path: /outpost.goauthentik.io
            pathType: Prefix
            backend:
            service:
                name: authentik-server
                port:
                number: 80
    tls:
    - hosts:
        - longhorn.mydomain.org
        secretName: longhorn-frontend-tls
    ```
2. Apply the manifest
    ```powershell
    kubectl apply -f longhorn-ingress.yaml
    ```

If everything is setup correctly when you navigate to https://longhorn.mydomain.org you will be redirected to Authentik to login before you can access the Longhorn UI.