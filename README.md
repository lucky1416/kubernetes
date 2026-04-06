# EKS Bootstrap Guide (AWS CLI, Terraform, kubectl, Ingress‑NGINX, cert‑manager)

**Tested on**: Ubuntu 24.04
**Region**: `ap-south-1`
**Cluster name**: `devopsshack-cluster`

---

## 1) Install AWS CLI v2

```bash
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt update && sudo apt install -y unzip
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Configure credentials (Access Key, Secret, region, output)
aws configure
```

> If you re-run the installer later, use `sudo ./aws/install --update`.

---

## 2) Install Terraform (HashiCorp APT repo)

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

curl -fsSL https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update && sudo apt-get install -y terraform
terraform -version
```

**Common init issue**

```bash
# If providers are locked to an older version, run:
terraform init -upgrade
```

---

## 3) Provision infra with Terraform

From your Terraform project directory (e.g., `~/Mega-Project-Terraform`):

```bash
terraform init
terraform apply --auto-approve
```

> Ensure variables (if any) are set (via `*.tfvars` or environment variables) before apply.

---

## 4) Install kubectl (latest stable)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Verify checksum (must show: kubectl: OK)
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

## 5) Generate kubeconfig for EKS

```bash
aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster

# Quick check (may take a minute after cluster creation)
kubectl get nodes
```

---

## 6) Install Ingress‑NGINX controller (v1.13.2)

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.2/deploy/static/provider/cloud/deploy.yaml

kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller

# Verify
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

> Once `EXTERNAL-IP`/hostname is assigned on the `ingress-nginx-controller` Service, note it for DNS.

---

## 7) Install cert‑manager (v1.19.0)

```bash
kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.19.0/cert-manager.yaml

# Wait for all three deployments
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-cainjector
kubectl -n cert-manager rollout status deploy/cert-manager-webhook

# Verify
kubectl -n cert-manager get pods
```

> Optional next step: create a ClusterIssuer (Let’s Encrypt HTTP‑01) and an Ingress with TLS annotations.

---

## 8) Quick verification cheatsheet

```bash
# Nodes ready?
kubectl get nodes -o wide

# Ingress controller LB hostname
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'; echo

# cert-manager webhook healthy?
kubectl -n cert-manager get deploy -o wide
```

---

## 9) Troubleshooting tips

* **ingress-nginx URL typos**: Avoid leading spaces or a stray backslash before `https://`.
* **No EXTERNAL-IP** on ingress LB: ensure cluster has public subnets with a route to an IGW, and Service type is `LoadBalancer`.
* **Terraform provider lock error**: `terraform init -upgrade`.
* **Auth errors**: `aws sts get-caller-identity` to confirm the right AWS account/role is active.

---

**Done.** You now have AWS CLI, Terraform, kubectl, EKS kubeconfig, Ingress‑NGINX, and cert‑manager installed and verified.
