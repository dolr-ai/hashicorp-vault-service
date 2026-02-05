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
```bash
# github-actions-policy.hcl

path "secret/data/*" {
  capabilities = ["read"]
}
```

NOTE : copy policy file into the container before writting policy on vault.

example:
```bash
docker cp /home/vault/policies/github-actions-policy.hcl vault:/tmp github-actions-policy.hcl

docker exec -it vault vault policy write github-actions /tmp/github-actions-policy.hcl
```




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

8. create approle for github action
```bash
cat <<EOF > github-role.json
{
  "role_type": "jwt",
  "bound_audiences": "https://github.com/dolr-ai",
  "bound_claims_type": "glob",
  "bound_claims": {
    "sub": "repo:dolr-ai/*:*"
  },
  "user_claim": "actor",
  "policies": "github-actions",
  "ttl": "1h"
}
EOF

docker cp github-role.json vault:/tmp/github-role.json
docker exec -it vault vault write auth/jwt/role/github-actions-role @/tmp/github-role.json
```

9. policy and secret can be set as explained in above section.