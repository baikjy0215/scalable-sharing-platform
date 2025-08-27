# ArgoCD Vault Plugin (AVP) Configuration
This document provides a guide for integrating Argo CD with HashiCorp Vault using the Argo CD Vault Plugin (AVP). This integration enables the secure management and injection of secrets from Vault into Kubernetes applications managed by ArgoCD.

## Overview
The Argo CD Vault Plugin (AVP) acts as a bridge between Argo CD and HashiCorp Vault. It allows Argo CD to retrieve secrets from Vault during the manifest rendering process and inject them into Kubernetes resources before they are applied to the cluster. This approach ensures that secrets are not stored in Git and are securely managed.

## Prerequisites
Before proceeding, ensure the following requirements are met:
- A running Kubernetes cluster
- kubectl installed and configured to communicate with the cluster
- A running instance of HashiCorp Vault that is accessible from the Kubernetes cluster

## Step 1: Install ArgoCD
First, install Argo CD in the Kubernetes cluster.

1. **Create a namespace for Argo CD**
```bash
kubectl create namespace argocd
```

2. **Apply the official Argo CD installation manifest**
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
For more installation options, refer to the [official documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/).

### Accessing the Argo CD UI
After installation, the ArgoCD UI can be accessed by port-forwarding the service.

1. **Forward the argocd-server service to a local port:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
The Argocd UI will be available at `https://localhost:8080`.

2. **Retrieve the initial admin password**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## Step 2: Configure the Argo CD Vault Plugin (AVP)
With Argo CD running, the next step is to configure the Argo CD Vault Plugin (AVP).

There are two ways to install the plugin as described in the [official documentation](https://argocd-vault-plugin.readthedocs.io/en/stable/installation/#initcontainer-and-configuration-via-sidecar). This guide uses the sidecar method as it offers more flexibility and is easier to manage.

### Using Sidecar Method
1. **Create the Plugin ConfigMap**

    The ConfigMap registers AVP as a Config Management Plugin for Helm, Kustomize, and standard Kubernetes manifests. An example ConfigMap can be found in [argocd-vault-plugin-cmp-plugin.yaml](files/argocd-vault-plugin-cmp-plugin.yaml).

2. **Create a Secret with the Vault configuration.**

    This secret will contain the Vault address, authentication method, and other necessary configurations. An example can be found in [argocd-vault-plugin-secret.yaml](files/argocd-vault-plugin-secret.yaml).

3. **Patch the Argo CD Repo Server**

   The final step is to patch argocd-repo-server Deployment to include the AVP container as a sidecar. This sidecar will process the manifests before they are applied to the cluster. An example can be found in [argocd-vault-plugin-sidecar.yaml](files/argocd-vault-plugin-sidecar.yaml).

## Usage in Application Manifests
With the plugin configured, secrets can be referenced in Kubernetes manifests using a placeholder syntax: \<path:secret/path#key>.

**Example: A Kubernetes Secret manifest**

Consider the following Secret manifest stored in a Git repository managed by Argo CD.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: <path:secret/data/my-app#username>
  password: <path:secret/data/my-app#password>
```

Upon deployment, the AVP sidecar will automatically fetch secrets from Vault and inject them into your application.