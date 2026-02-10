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

3. Fetch passwords in your github-action:
- ***role** will be set for repository, so that repo can be access only secrets which are related to repo.

```yml
- name: Import Secrets from Vault
        uses: hashicorp/vault-action@v3
        with:
          url: https://vault.yral.com
          method: jwt
          role: <REPO_NAME>-role
          # Optional: customize the audience
          jwtGithubAudience: "https://github.com/dolr-ai"
          secrets: |
            secret/data/<REPO_NAME>/config test_pass | DB_PASSWORD ;
            secret/data/<REPO_NAME>/config test_user | DB_USERNAME ;
```
