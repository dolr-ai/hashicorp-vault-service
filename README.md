# hashicorp-vault-service

This project uses **HashiCorp Vault** for centralized secret management and **GitHub Actions OIDC (JWT)** for secure, short-lived authentication from CI pipelines.

✅ Admins can provide access to repo to fetch secret, provide shared secret access to repo, create common secrets and UI access to dev and admin.  
✅ Developers can **fetch** secrets in GitHub Actions  
❌ NO one can see raw secret values  

## Process Overview

1. **Admin** writes secrets into Vault..
2. KV secret engine (kv-v2) has been enable to store secret securely
3. Vault has a **policies** that allows read-only access to specific paths.
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
- **role** will be set for repository, so that repo can be access only secrets which are related to repo.
- if you want to add shared secret ask admin to add shared secret to your repo.

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
            # To read repo based secrets
            secret/data/<REPO_NAME>/prod db_password | DB_PASSWORD ;
            secret/data/<REPO_NAME>/prod db_user | DB_USERNAME ;

            # To read shared secrets
            secret/data/shared-secret-name value | SHARED_SECRET ;
```
