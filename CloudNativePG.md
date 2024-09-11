# CloudNativePG

This will guide you on setting up [CloudNativePG](https://cloudnative-pg.io/) in Kubernetes to create a 3 node HA Postgres cluster.

## Install the Operator

1. Setup the helm repo

    ```powershell
    helm repo add cnpg https://cloudnative-pg.github.io/charts
    helm repo update
    ```

2. Install the helm chart

    ```powershell
    helm upgrade --install cnpg `
      --namespace cnpg-system `
      --create-namespace `
      cnpg/cloudnative-pg
    ```

## Create a Postgres Cluster

Now that the operator is installed lets creeate a 3 node cluster callled `postgres-cluster`.

1. Create a file called `postgres-cluster.yaml`

    ```yaml
    apiVersion: postgresql.cnpg.io/v1
    kind: Cluster
    metadata:
      name: postgres-cluster
    spec:
      instances: 3
      enableSuperuserAccess: true
      storage:
        size: 1Gi
    ```
    Read [here](https://cloudnative-pg.io/documentation/1.20/cloudnative-pg.v1/#postgresql-cnpg-io-v1-ClusterSpec) to learn more about the different options.

2. Apply the manifest

    ```powershell
    kubectl apply -f postgres-cluster.yaml
    ```

## Cluster Access

Once the cluster is setup 3 different services are automaticly created in the cluster that start with the cluster name e.g. `postgres-cluster`

- **postgres-cluster-rw**: Points to the primary instance of the cluster (read/write).
- **postgres-cluster-ro**: Points to the replicas, where available (read-only).
- **postgres-cluster-r**: Points to any PostgreSQL instance in the cluster (read).

When setting up an application to point to the service use `postgres-cluster-rw.default.svc.cluster.local`

## External Access

Now lets expose the cluster to the LAN and connect to to it using [pgAdmin](https://www.pgadmin.org/download/pgadmin-4-windows/)

1. Create a file called `postgres-cluster-service.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres-cluster-external
      namespace: default
    spec:
      type: LoadBalancer
      loadBalancerIP: 172.16.8.13
      ports:
      - name: postgres
          port: 5432
          protocol: TCP
          targetPort: 5432
      selector:
          cnpg.io/cluster: prod-pg-cluster
          role: primary
    ```
    Make sure to change the `loadBalancerIP` to something in your MetalLB IP Range.

2. Apply the manifest
    ```powershell
    kubectl apply -f postgres-cluster-service.yaml
    ```

3. Lets get the postgres admin password
    1. Log into Rancher
    2. Goto [Local Cluster] -> Storage -> Secrets
    3. Look for a secret called `postgres-cluster-superuser` and go into it
    4. Copy the password.

4. Open [pgAdmin](https://www.pgadmin.org/download/pgadmin-4-windows/)
5. Register a new server
    - Name: `postgres-cluster`
    - Host name/address: `172.16.8.13`
    - port: `5432`
    - username: `postgres`
    - password: copied from the step above

You should now be connected to the postgres cluster that you created using the CloudNativePG Operator.


## Links

- [Quick Start Guid](https://cloudnative-pg.io/documentation/1.16/quickstart/)
- [Documentation](https://cloudnative-pg.io/documentation/1.24/)
- [Github Repo](https://github.com/cloudnative-pg/cloudnative-pg/)