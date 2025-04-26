# 🚀 THORNode Deployment on Linode (Kubernetes)

A complete, end‑to‑end cookbook for standing up a **THORNode validator or fullnode** on **Linode Kubernetes Engine (LKE)** and operating it from an Ubuntu workstation.

---

## 1  Create a Linode Personal‑Access‑Token (PAT)

Generate a PAT with **Read/Write** scope so automation tools can create resources in your account.

1. Log in to **cloud.linode.com** ➜ avatar ➜ **API Tokens**.  
2. Click **Create a Personal Access Token**, pick an expiry (e.g., 90 days), grant full RW scope, then **Create Token**.  
3. **Copy the token once and store it securely** (the UI won’t show it again).

---

### 🔄 Token rotation every 3 months

Keeping your PAT short‑lived shrinks the window an attacker could abuse it if the secret leaks.

**Recommended cadence:** create a replacement ~15 days before each expiry, swap it into all scripts, then revoke the old one.

#### Step 1  Create the next 90‑day PAT

```bash
curl -sS -H "Authorization: Bearer $OLD_PAT"      -H "Content-Type: application/json"      -d "{
           \"label\": \"thornode-$(date +%Y%m%d)\",
           \"expiry\": \"$(date -d \"+90 days\" --iso-8601=seconds)\",
           \"scopes\": \"*\"
         }"      https://api.linode.com/v4/profile/tokens
```

Copy the `token` value from the JSON response.

#### Step 2  Export the new token for this shell

```bash
export LINODE_CLI_TOKEN="PASTE_NEW_TOKEN"
```

#### Step 3  Verify the new PAT works

```bash
linode-cli --json linodes list | jq length
```

#### Step 4  Revoke the expiring token (optional)

```bash
linode-cli --json profile tokens-list | jq      # find OLD_TOKEN_ID
linode-cli profile tokens-revoke OLD_TOKEN_ID
```

---

## 2  Set up your **Ubuntu** workstation

Install all command‑line tools needed to build and manage the cluster.

### 2.1  Update APT and utilities

```bash
sudo apt-get update
sudo apt-get install -y wget curl jq gnupg software-properties-common
```

### 2.2  Install Terraform (HashiCorp repo)

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg   | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main"   | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update
sudo apt-get install -y terraform
```

### 2.3  Install kubectl (new `pkgs.k8s.io` repo)

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key   | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.asc > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.asc] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /"   | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
```

### 2.4  Install Linode CLI (pipx – PEP 668 friendly)

```bash
sudo apt-get install -y pipx
pipx ensurepath
pipx install linode-cli
```

### 2.5  Configure Linode CLI

```bash
linode-cli configure --token "PASTE_YOUR_PAT"
```

| Prompt | Recommended input |
|--------|-------------------|
| Default Region | `us-east` (or nearest) |
| Default Type of Linode | *Enter* |
| Default Image | *Enter* |
| Custom API target | `N` |
| Suppress API Version Warnings | `N` |

---

## 3  Clone THORChain’s **cluster‑launcher**

```bash
git clone https://gitlab.com/thorchain/devops/cluster-launcher
cd cluster-launcher
```

---

## 4  Provision the Kubernetes cluster on LKE

### Terraform path

```bash
cd linode
# create terraform.tfvars with token, region, etc.
terraform init
terraform apply -auto-approve
```

*Example `terraform.tfvars`:*

```hcl
token           = "YOUR_PAT"
region          = "us-east"
cluster_label   = "thornode"
cluster_version = "1.32"

pool_settings = {
  desired_capacity = 3
  min_capacity     = 3
  max_capacity     = 10
  instance_type    = "g6-standard-4"
}
```

> **Important:** after Terraform finishes, run:
>
> ```bash
> cd ..
> SHELL=/bin/bash make linode-post
> ```

---

## 5  Pull the kubeconfig, merge it locally, and sanity‑check the cluster

```bash
# kubeconfig was saved by Terraform to:
#   ~/.kube/config-linode

# 5.1 Merge with any existing configs
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/config-linode
kubectl config view --flatten > $HOME/.kube/tmp
mv $HOME/.kube/tmp $HOME/.kube/config

# 5.2 Switch kubectl to the new context
kubectl config use-context thornode   # or your chosen label

# 5.3 Smoke-test: all nodes should be Ready
kubectl get nodes -o wide
```

---

## 6  Install THORNode & helper tooling

```bash
git clone https://gitlab.com/thorchain/devops/node-launcher
cd node-launcher
make helm
make tools                       # optional monitoring stack
NET=mainnet TYPE=validator NAME=thornode make install
```

---

## 7  Discover your public endpoints

```bash
kubectl get svc
```

---

## 8  Open monitoring dashboards

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &
xdg-open http://localhost:3000
```

---

## 9  Clean‑up

```bash
make destroy-linode   # or terraform destroy
```
