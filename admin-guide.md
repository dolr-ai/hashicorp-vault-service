# Admin Guide — Storing Secrets in Vault

### Write a Secret

- need to ssh into one of the vault node, login with vault-token and put secrets at particular path (can set multiple secrets at one path)

- login into the vault with root token.
```bash
docker exec -it vault vault login <ROOT_TOKEN>
```

Example: Store application database credentials
```bash
docker exec -it vault vault kv put secret/test-app/config \
  db_username="appuser" \
  db_password="super-secret-password"
```

### Create policy and grant access to repo
- admin can create policy and prove access of particular repo related secret to repo using repo_to_vault_access_provider.yml .
- it will take repo name as input and we can select grant/revoke action.
- as example below listed policy will be created for repo and repo related role will be created with this policy

policy-example:
```hcl
# hot-or-not-web-leptos-ssr-policy.hcl

path "secret/data/hot-or-not-web-leptos-ssr/*" {
  capabilities = ["read", "list"]
}
```

role-example:
```json
{
   "role_type": "jwt",
   "bound_audiences": "https://github.com/dolr-ai",
   "bound_claims_type": "glob",
   "bound_claims": {
     "repository": "dolr-ai/hot-or-not-web-leptos-ssr"
   },
   "user_claim": "actor",
   "policies": ["hot-or-not-web-leptos-ssr"],
   "ttl": "1h"
}

```

- developers can fetch secrets in their CI using this role which is dedicated for specific repo and can only fetch secret related to repo.


### Create shared-secret
- by using create_common_secret.yml , admin can create common secrets which will on their distinct path and also it will read access policy for that shared secret.
- this yml will take shared secret name and value as input and write it on vault and creates associated policy.

### Manage shared-secret access
- by using manage_shared_secret_access_to_repo.yml , with this ymladmin can append read policy to that of that particular shared secret to repo role.
- this yml will take repo name and shared secret name as input and we can set grant/revoke action.

### UI access provision
- by using ui_access_provider.yml , admin can create admin and dev user to acces vault-ui. 
- admin has to pass user email and name of role. Also, has to select role type and grant/revoke actionfrom drop down.



# Vault-cluster setup : 
(NOTE: no need to run it cluster is already up and running)

1. Run **deploy-vault.yml** from workflow, it will create 3 node vault cluster and start vault containers.

2. Generate custom TLS certs for the cluster nodes and upload them.

3. Initialise vault on of the node. this step will create 5 unseal keys and a root token, store it safely.
```bash
docker exec -it vault vault operator init
```

4. Unseal the all vaults one after one, need to provide 3 unseal keys to each node, provide each key one after one.
```bash
docker exec -it vault vault operator unseal <UNSEAL_KEY>
```

5. Upon successful unseal there will be one leader and two followers. Can check with this command, but before that we need to login to the vault as well with root token.
```bash
docker exec -it vault vault login <ROOT_TOKEN>
docker exec -it vault vault operator raft list-peers
```

6. enable secret engine and verify initialisation.
```bash
docker exec -it vault vault secrets enable -path=secret kv-v2
docker exec -it vault vault secrets list
```

7. enable jwt auth for github
```bash
docker exec -it vault vault auth enable jwt
docker exec -it vault vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"
```

8. enable oidc auth for UI access with gmail
```bash
docker exec -it vault vault auth enable oidc
docker exec -it vault vault write auth/oidc/config \
  oidc_discovery_url="https://accounts.google.com" \
  oidc_client_id="SOME_CLIENT_ID" \
  oidc_client_secret="SOME_CLIENT_SECRET" 
```

9. manually create policy and role for vault-repo first to access all functinalities through yml.
- create policy for vault repo
```hcl
#vault-repo-policy.hcl
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

```bash
#create a vault-repo-policy file and copy it into container (This policy allow hashicorp-vault-service repo to create and remove access for repos and admins)
docker cp vault-repo-policy.hcl vault:/tmp/vault-repo-policy.hcl
docker exec -it vault vault policy write vault-service /tmp/vault-repo-policy.hcl

//verify policy
docker exec -it vault vault policy read vault-service
```


- create role for vault repo
```json
# vault-service-role.json
{
   "role_type": "jwt",
   "bound_audiences": "https://github.com/dolr-ai",
   "bound_claims_type": "glob",
   "bound_claims": {
     "repository": "dolr-ai/hashicorp-vault-service"
   },
   "user_claim": "actor",
   "policies": "vault-service",
   "ttl": "1h"
}
```
```bash
docker cp vault-service-role.json vault:/tmp/vault-service-role.json
docker exec -it vault vault write auth/jwt/role/vault-service-role @/tmp/vault-service-role.json

//verify role
docker exec -it vault vault read auth/jwt/role/vault-service-role-role
```