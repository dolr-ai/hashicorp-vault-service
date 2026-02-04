# hashicorp-vault-service

This project uses **HashiCorp Vault** for centralized secret management and **GitHub Actions OIDC (JWT)** for secure, short-lived authentication from CI pipelines.

✅ Admins manually store secrets in Vault  
✅ Developers can *use* secrets in GitHub Actions  
❌ NO one can see raw secret values  

## Architecture Overview

1. **Admin** writes secrets into Vault..
2. KV secret engine (kv-v2) has been enable to store secret securely
3. Vault has a **policy** that allows read-only access to specific paths (currently github-action policy has been set which enables github-actions fetch secrets from whole secret database, no other policy has been set so no one can read directly from vault).
4. GitHub Actions authenticates to Vault using **OIDC JWT** (no static tokens).
5. Vault validates the GitHub identity and returns a **temporary token**.
6. Workflow fetches secrets securely at runtime.


## 👨‍💻 Developer Guide — Fetching Secrets in GitHub Actions

1. Workflow permission:
Your workflow must allow OIDC:

```yml
permissions:
  id-token: write
  contents: read
```

2. Add **VAULT_CA_CERT** secret to repo secrets:
- VAULT_CA_CERT required to connect with vault via tls secured connection. 

3. Fetch passwords in your github-action:
- 
```yml
- name: Import Secrets from Vault
    uses: hashicorp/vault-action@v3
    with:
        url: https://vault.example.com
        method: jwt
        role: github-actions-role
        jwtGithubAudience: "https://github.com/dolr-ai"
        # keep it false to verify CA certificate
        tlsSkipVerify: false
        caCertificate: "${{ secrets.VAULT_CA_CERT }}"
        secrets: |
        secret/data/test-app/config test_pass | DB_PASSWORD ;
        secret/data/test-app/config test_user | DB_USERNAME ;
```
