# 🔐 HashiCorp Vault – Production-Style Setup (Single Node)

This guide documents a **production-style Vault setup on a single VM** for learning and future scaling to a cluster.

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

# 🔐 3. Set Environment Variables

```bash
export VAULT_ADDR="https://192.168.56.13:8200"
export VAULT_SKIP_VERIFY=true   # temporary for self-signed cert
```

---

# 🔑 4. Initialize Vault

```bash
vault operator init
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

Test:

```bash
vault kv put secret/myapp username=admin password=123
vault kv get secret/myapp
```

---

# 🛡️ 8. Create Policy (RBAC)

Create file `app-policy.hcl`:

```hcl
path "secret/data/myapp" {
  capabilities = ["read"]
}
```

Apply:

```bash
vault policy write app-policy app-policy.hcl
```

---

# 👤 9. Enable Authentication (Userpass)

```bash
vault auth enable userpass
```

Create user:

```bash
vault write auth/userpass/users/admin \
  password=123 \
  policies=app-policy
```

---

# 🌐 10. Login via UI

Open:

```
https://192.168.56.13:8200
```

Use:

* Method: userpass
* Username: admin
* Password: 123

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

# 📌 Author Note

This setup is designed for **learning production concepts** and can be extended to a full cluster without reinstallation.

---