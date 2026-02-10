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

### Set Policy
- admin can set policy for read write to particular path to particular approle.
```hcl
# hot-or-not-web-leptos-ssr-policy.hcl

path "secret/data/hot-or-not-web-leptos-ssr/*" {
  capabilities = ["read", "list"]
}
```

NOTE : copy policy file into the container before writting policy on vault.

example:
```bash
docker cp /home/vault/policies/hot-or-not-web-leptos-ssr-policy.hcl vault:/tmp/hot-or-not-web-leptos-ssr-policy.hcl

docker exec -it vault vault policy write hot-or-not-web-leptos-ssr /tmp/hot-or-not-web-leptos-ssr-policy.hcl
```

### Set Role
- admin need to create a role for each repo with dedicated policy for that repo.

- first create role json in roles directory
example:
```json
{
   "role_type": "jwt",
   "bound_audiences": "https://github.com/dolr-ai",
   "bound_claims_type": "glob",
   "bound_claims": {
     "repository": "dolr-ai/hot-or-not-web-leptos-ssr"
   },
   "user_claim": "actor",
   "policies": "hot-or-not-web-leptos-ssr",
   "ttl": "1h"
}

```
- copy to container and write role to vault.
example:
```bash
docker cp hot-or-not-web-leptos-ssr-role.json vault:/tmp/hot-or-not-web-leptos-ssr-role.json

docker exec -it vault vault write auth/jwt/role/hot-or-not-web-leptos-ssr-role @/tmp/hot-or-not-web-leptos-ssr-role.json
```

- developers can fetch secrets in their CI using this role which is dedicated for specific repo and can only fetch secret related to repo.




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

8. create set policy and approle as directed above.