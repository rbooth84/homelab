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