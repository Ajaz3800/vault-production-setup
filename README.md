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
ClusterSecretStore (ESO)
        ↓
ExternalSecret
        ↓
Kubernetes Secret
        ↓
Pod/Application
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
---

# 🔐 5. Configure Kubernetes Auth

* For Cluter IP
```bash
kubectl cluster-info
```
* Create Service Account Tocken
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

# 🔗 9. Create ClusterSecretStore

## Why?

This connects Kubernetes to Vault.

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
```

```bash
kubectl apply -f clustersecretstore.yaml
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

# 🎯 Final Result

* Secret stored in Vault
* Automatically synced to Kubernetes
* No hardcoded secrets in YAML

---


# 🤖 11. Enable AppRole (CI/CD)

```bash
vault auth enable approle
```

```bash
vault write auth/approle/role/my-role \
  token_policies="app-policy"
```

---

# 📊 12. Enable Audit Logging (IMPORTANT)

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

# 🔍 13. Verify Audit Logs

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

---

# 🔒 Security Best Practices

* Never use root token in daily work
* Use policies (RBAC)
* Enable audit logging
* Use TLS (HTTPS)
* Restrict firewall access
* Backup `/opt/vault/data`

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

---

# 🎯 Final Summary

This setup provides:

✅ Secure Vault with TLS
✅ Production-style storage (Raft)
✅ Authentication & RBAC
✅ Audit logging
✅ CI/CD ready (AppRole)

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