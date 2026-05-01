# 🔐 Vault + Kubernetes External Secrets (ESO) Full Setup Guide

This document provides a complete, step-by-step guide to integrate:

HashiCorp Vault
Kubernetes Authentication
External Secrets Operator (ESO)

It includes:

CLI steps
YAML configurations
Clear explanations of why each component is required

---

# 📌 Architecture Overview

```
Vault (KV Secrets Engine)
        ↓
Kubernetes Auth (ServiceAccount)
        ↓
Vault Role + Policy
        ↓
ClusterSecretStore (ESO)        SecretStore (ESO)
(Cluster-wide access)           (Namespace-scoped access)
        ↓                               ↓
ExternalSecret                  ExternalSecret
        ↓                               ↓
Kubernetes Secret               Kubernetes Secret
        ↓                               ↓
Pod/Application                 Pod/Application
```

---

# 📌 Overview

This setup includes:

* TLS-enabled Vault
* Raft storage (production-grade)
* Audit logging
* Authentication (userpass + AppRole)
* Policies (RBAC)
* Production best practices

⚠️ Note: This is a **single-node setup (not HA)** but follows **production standards**.

---

# 🛠️ Install HashiCorp Vault

Follow the official installation guide:

👉 https://developer.hashicorp.com/vault/install

---

# 🖥️ Environment

* OS: Ubuntu VM
* Vault IP: `192.168.56.13`
* Vault Port: `8200`

---



# ⚙️ 1. Vault Configuration (`vault.hcl`)

```hcl
ui = true

api_addr     = "https://192.168.56.13:8200"
cluster_addr = "https://192.168.56.13:8201"

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node1"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"

  tls_cert_file = "/opt/vault/tls/tls.crt"
  tls_key_file  = "/opt/vault/tls/tls.key"
}

disable_mlock = true
```

---

# 🚀 2. Start Vault

```bash
vault server -config=/etc/vault.d/vault.hcl
```

---

# Self-Signed Certificate (dev only)

```bash
# 1. Create the configuration file
cat > vault-san.cnf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = 192.168.56.13

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.56.13
EOF

# 2. Generate the new Key and Certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/vault/tls/tls.key \
  -out /opt/vault/tls/tls.crt \
  -config vault-san.cnf -extensions v3_req

# 3. Set correct permissions
sudo chown vault:vault /opt/vault/tls/tls.key /opt/vault/tls/tls.crt
sudo chmod 640 /opt/vault/tls/tls.key /opt/vault/tls/tls.crt

#. set files permissions
sudo chown vault:vault /opt/vault/data/node-id /opt/vault/data/vault.db
sudo chmod 640 /opt/vault/data/node-id /opt/vault/data/vault.db

# 5. Restart Vault
sudo systemctl restart vault
```

# 🔐 3. Set Environment Variables

```bash
export VAULT_ADDR="https://192.168.56.13:8200"
export VAULT_SKIP_VERIFY=true   # temporary for self-signed cert
```

---

# 🔑 4. Initialize Vault
After starting Vault for the first time, you must initialize it.

```bash
vault operator init -format=json > vault-cluster-vault-init.json
```

Save:

* Unseal Keys (IMPORTANT)
* Root Token

---

# 🔓 5. Unseal Vault

```bash
vault operator unseal
```

(Enter 3 unseal keys)

---

# 🔑 6. Login

```bash
vault login <ROOT_TOKEN>
```

---

# 📦 7. Enable KV Secrets Engine

```bash
vault secrets enable -path=secret kv-v2
```

---

# Store Secret in Vault

```bash
vault kv put secret/devsecops-gitops sshPrivateKey="test123"
```

# 👤 Vault Userpass Authentication with Admin Policy

This section explains how to create an admin user using Userpass auth method and assign a full access admin policy in HashiCorp Vault.

# 🔐 1. Create Admin Policy

Create policy file:
```bash
nano admin-policy.hcl
```
Paste the following:
```bash
# ADMIN POLICY - FULL ACCESS (USE CAREFULLY)
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
```
Apply policy:
```bash
vault policy write admin-policy admin-policy.hcl
```
# 👤 2. Enable Userpass Authentication
```bash
vault auth enable userpass
```
⚠️ If already enabled, ignore the error.

# 👤 Create Admin User:

```bash
vault write auth/userpass/users/admin \
    password="Admin@123" \
    policies="admin-policy"
```

---

# Login via UI

Open:

```
https://<YOUR IP>:8200
```

Use:

* Method: userpass
* Username: admin
* Password: 123

---

# ☸️ 3. Enable Kubernetes Auth Method

```bash
vault auth enable kubernetes
```

---

# 🧑‍💻 4. Create Kubernetes Service Account

```bash
nano sa.yaml
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: vault
```

```bash
kubectl apply -f sa.yaml
```
### Create Secret for token
```bash
nano vault-auth-token.yaml
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-token
  namespace: vault
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
```
```bash
kubectl apply -f vault-auth-token.yaml
```
```bash
kubectl get secret vault-auth-token -n vault -o jsonpath='{.data.token}' | base64 -d
```
---

# 🔐 5. Configure Kubernetes Auth

* For Cluter IP
```bash
kubectl cluster-info
```
* Create Service Account Token
```bash
kubectl create token vault-auth -n vault
```

* Export Kubernetes ca.crt
```bash
echo <certificate-authority-data> | base64 -d
```

```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://<API-SERVER>:6443" \
  kubernetes_ca_cert=@ca.crt \
  token_reviewer_jwt="<jwt>"
```


---

# 📜 6. Create Vault Policy

```bash
nano policy.hcl
```

```hcl
path "secret/data/devsecops-gitops" {
  capabilities = ["read"]
}
```


```bash
vault policy write devsecops-policy policy.hcl
```

---

# 🔑 7. Create Kubernetes Auth Role

```bash
vault write auth/kubernetes/role/devsecops-role \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=vault \
  policies=devsecops-policy \
  ttl=1h
```

---

# 📦 8. Install External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```
---

# 🔗 9. Create ClusterSecretStore (For Single team evirment)

## Why?

ClusterSecretStore is cluster-scoped. It allows any namespace in the cluster to reference it and fetch secrets from Vault. Best for platform/DevOps teams managing secrets centrally.

```bash
nano clustersecretstore.yaml
```

* Optional
```bash
sudo cat /opt/vault/tls/tls.crt | base64 -w 0
```

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://<VAULT-IP>:8200"
      path: "secret"
      version: "v2"
      #caBundle: ""
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "devsecops-role"
          serviceAccountRef:
            name: vault-auth
            namespace: vault
          secretRef:
            name: vault-auth-token
            namespace: vault
            key: token  
```

```bash
kubectl apply -f clustersecretstore.yaml
```
```bash
kubectl get clustersecretstore
```

---

# 🔐 10. Create ExternalSecret

## Why?

This fetches data from Vault and creates Kubernetes Secret.
```bash
nano externalsecret.yaml
```
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: vault
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: my-k8s-secret
    creationPolicy: Owner
  data:
    - secretKey: sshPrivateKey
      remoteRef:
        key: devsecops-gitops
        property: sshPrivateKey
```

```bash
kubectl apply -f externalsecret.yaml
```

---

# ✅ Verification

```bash
kubectl get externalsecret -n vault
kubectl get secret my-k8s-secret -n vault -o yaml
```

Expected:

```
READY: True
```

---

# 🏠 11. Create SecretStore (Namespace-Scoped)
### Why SecretStore?
### Unlike `ClusterSecretStore`, a `SecretStore` is namespace-scoped. It can only be used by `ExternalSecrets` within the same namespace. This is ideal for:
- Multi-tenant clusters where teams need isolation
- App-level secret management with dedicated Vault roles
- Demonstrating least-privilege access per namespace


> 💡 Both `ClusterSecretStore` and `SecretStore` can run simultaneously in the same cluster.

---

# Step 1 — Create a Dedicated Namespace
```bash
kubectl create namespace dev
```
---
# Step 2 — Create ServiceAccount in `dev` Namespace
```bash
nano sa-dev.yaml
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: dev
```
```bash
kubectl apply -f sa-dev.yaml
```
### Create Static Token Secret for `dev` SA
> ⚠️ Always use a static token secret (not kubectl create token) — short-lived tokens expire and cause 403 errors.

```bash
nano vault-auth-token-dev.yaml
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-token
  namespace: dev
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
```
```bash
kubectl apply -f vault-auth-token-dev.yaml
```
```bash
kubectl get secret vault-auth-token -n dev -o jsonpath='{.data.token}' | base64 -d
```

# Step 3 — Store a Secret in Vault for dev

```bash
vault kv put secret/dev-app db_password="SuperSecret123"
```

# Step 4 — Create Vault Policy for dev
```bash
nano dev-policy.hcl
```
```
path "secret/data/dev-app" {
  capabilities = ["read"]
}
```
```bash
vault policy write dev-policy dev-policy.hcl
```

# Step 5 — Create Vault Role for dev Namespace

```bash
vault write -address="https://192.168.56.13:8200" -tls-skip-verify \
  auth/kubernetes/role/dev-role \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=dev \
  policies=dev-policy \
  ttl=1h
```

# Step 6 — Create SecretStore YAML

```bash
nano secretstore.yaml
```
```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault-backend-local
  namespace: dev
spec:
  provider:
    vault:
      server: "https://192.168.56.13:8200"
      path: "secret"
      version: "v2"
      caBundle: "<same-caBundle-as-ClusterSecretStore>"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "dev-role"
          serviceAccountRef:
            name: vault-auth
          secretRef:
            name: vault-auth-token
            key: token
```
```bash
kubectl apply -f secretstore.yaml
```
```bash
kubectl get secretstore -n dev
```

### Expected output:
```
NAME                  AGE   STATUS   CAPABILITIES   READY
vault-backend-local   10s   Valid    ReadWrite      True
```

# Step 7 — Create ExternalSecret using SecretStore
```bash
nano externalsecret-dev.yaml
```
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: dev-secret
  namespace: dev
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend-local   
    kind: SecretStore        
  target:
    name: dev-app-secret
    creationPolicy: Owner
  data:
    - secretKey: db_password
      remoteRef:
        key: dev-app         
        property: db_password
```

---

# ✅ Verification (SecretStore)

```bash
kubectl get secretstore -n dev
```
```bash
kubectl get externalsecret -n dev
```
```bash
kubectl get secret dev-app-secret -n dev -o yaml
```
```bash
kubectl get secret dev-app-secret -n dev -o jsonpath='{.data.db_password}' | base64 -d
```

# Expected:
```
READY: True
SuperSecret123
```

# 🔍 ClusterSecretStore vs SecretStore — Quick Comparison
```
Feature             ClusterSecretStore        SecretStore
-------------------------------------------------------------------------
Scope               Cluster-wide              Single namespace
Who can use it      All namespaces            Same namespace only
Vault Role Binding  One role for all          One role per namespace
Best for            Platform / DevOps teams   App teams / multi-tenant
Isolation           Lower                     Higher
Management overhead Single config             One per namespace
```
> 🎯 Portfolio Tip: Running both simultaneously shows you understand centralized vs decentralized secrets management — a strong DevSecOps skill.

---

# 🤖 12. Enable AppRole (CI/CD)

```bash
vault auth enable approle
```

```bash
vault write auth/approle/role/my-role \
  token_policies="app-policy"
```

---

# 📊 13. Enable Audit Logging (IMPORTANT)

## Fix permissions:

```bash
sudo touch /var/log/vault_audit.log
sudo chown vault:vault /var/log/vault_audit.log
sudo chmod 640 /var/log/vault_audit.log
```

## Enable audit:

```bash
vault audit enable file file_path=/var/log/vault_audit.log
```

---

# 🔍 14. Verify Audit Logs

```bash
vault kv put secret/test foo=bar
cat /var/log/vault_audit.log
```

---

# ⚠️ Common Issues & Fixes

## ❌ TLS Error (127.0.0.1 mismatch)

```
x509: cannot validate certificate
```

✅ Fix:

```bash
export VAULT_ADDR="https://192.168.56.13:8200"
```

---

## ❌ Permission Denied (Audit log)

```
permission denied: /var/log/vault_audit.log
```

✅ Fix:

```bash
sudo chown vault:vault /var/log/vault_audit.log
```

---

## ❌ User Login Failed

```
invalid username or password
```

✅ Fix:

* Create user first
* Enable `userpass` auth

---

## ❌ sudo removes env variables

✅ Fix:

```bash
sudo -E vault <command>
```

### ❌ ClusterSecretStore / SecretStore 403 Permission Denied

```
unable to log in with Kubernetes auth: Error making API request.
Code: 403. Errors: permission denied
```

## ✅ Fix:
- Never use `kubectl create token` — tokens expire and cause 403 errors
- Always use a static `kubernetes.io/service-account-token` Secret
- Verify SA name and namespace match exactly what is in the Vault role:
```bash
vault read auth/kubernetes/role/<role-name>
```
- Reconfigure Vault Kubernetes auth with the static token:
```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://<API-SERVER>:6443" \
  token_reviewer_jwt="$(kubectl get secret vault-auth-token -n <namespace> \
    -o jsonpath='{.data.token}' | base64 -d)" \
  kubernetes_ca_cert="$(kubectl get secret vault-auth-token -n <namespace> \
    -o jsonpath='{.data.ca\.crt}' | base64 -d)"
```

---

# 🔒 Security Best Practices

* Never use root token in daily work
* Use policies (RBAC)
* Enable audit logging
* Use TLS (HTTPS)
* Restrict firewall access
* Backup `/opt/vault/data`
* Always use static SA token secrets (not short-lived tokens)
* Use SecretStore per namespace for multi-tenant isolation

---

# 🚀 Next Steps (Production Upgrade)

* Convert to **3-node Vault cluster**
* Enable **Auto-Unseal (AWS KMS / etc)**
* Integrate with **Kubernetes**
* Use **External Secrets Operator**

---

# 🧠 Key Learnings

* Vault does NOT have default users
* TLS must match IP (SAN issue)
* Vault server runs as `vault` user (permission matters)
* Raft storage is required for production
* Audit logging is critical
* kubectl create token generates short-lived tokens — always use static SA token secrets
* ClusterSecretStore = platform-level; SecretStore = app-level

---

# 🎯 Final Summary

This setup provides:

✅ Secure Vault with TLS<br>
✅ Production-style storage (Raft)<br>
✅ Authentication & RBAC<br>
✅ Audit logging<br>
✅ CI/CD ready (AppRole)<br>
✅ ClusterSecretStore (cluster-wide secrets)<br>
✅ SecretStore (namespace-scoped secrets)<br>

❌ Not HA (single node)

---

## ⭐ Support

If you find this project helpful, please give it a star ⭐ on GitHub.

---

## 🌐 Connect With Me

<div align="center">
  
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/shaikh-muhammad-ajaz)
[![Email](https://img.shields.io/badge/Email-shaikhajaz38000@gmail.com-red?style=for-the-badge&logo=gmail&logoColor=white)](mailto:shaikhajaz38000@gmail.com)
[![YouTube](https://img.shields.io/badge/YouTube-Subscribe-red?style=for-the-badge\&logo=youtube\&logoColor=white)](https://www.youtube.com/@devopswithajaz)
</div>

<div align="center">

[![Upwork](https://img.shields.io/badge/Upwork-Hire%20Me-6FDA44?style=for-the-badge&logo=upwork&logoColor=white)](https://upwork.com/freelancers/muhammadajaz)
[![Fiverr](https://img.shields.io/badge/Fiverr-Order%20Now-1DBF73?style=for-the-badge&logo=fiverr&logoColor=white)](https://www.fiverr.com/ajazshaikh3800)
</div>

---

<div align="center">
  
### 💡 "Turning ideas into production-ready systems."

![Profile Views](https://komarev.com/ghpvc/?username=Ajaz3800&color=brightgreen&style=flat-square)
[![GitHub followers](https://img.shields.io/github/followers/Ajaz3800?label=Follow&style=social)](https://github.com/Ajaz3800)

</div>