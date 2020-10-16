# HashiCorp Vault
### Manage Secrets and Protect Sensitive Data : https://www.vaultproject.io/

Secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API.

### Authentication methods :
- For human users : LDAP, Active Directory, AWS Identity IAM, GitHub personal tokens
- For applications : AppRole

### Tokens
- User need first to authenticate to get a token
- Then he uses the token to request the secret key
- Tokens has a TTL and / or a limited number of uses

## How to setup a dev server
Not to be used in prod, insecure !
```shell script
vault server -dev   # display the server config + unseal key + root token
export VAULT_ADDR = http://127.0.0.1:8200
vault status
vault kv put secret/MySecret password=myPassword
vault kv get [-format=json] secret/MySecret
```

## Secret engines
##### Build in secret engines :
- Key value KV
- Cloud : Azure, AWS, GCP
- Github
- SSH
- Database
- Token

##### Enable a secret engine + write to the associated path
```shell script
vault secret enable database
vault write database/config/mysql_app1
```
```shell script
vault secrets enable -path=myAppDB database
vault write myAppDB/config/mysql_app1
```

##### Use a secret engine
- List : `vault secrets list`
- Enable : `vault secrets enable -path=myApp kv`
- Add a key : `vault kv put myApp/mySecret key=value`
- Help : `vault path-help $engine_type`
  where `$engine_type` can be kv, database, ...
  
###### Authentication
- Servers running : `vault status`
- Enabled credentials : `vault auth list`
- `vault auth enable`
- `vault write auth/userpass/users/vaultuser password=vault`
- generate a token with a TTL : `vault login -method=userpass username=vaultuser password=vault`  
- Login : `vault login $token`
- `vault token create [-ttl=5m]`
  > Generate a token with the same privileges as the last login
  > Optionnally set a TTL : 768h by default

##### Accessors
When tokens are created, a token accessor is also created  
This accessor can be used to perform limited actions
- Display accessors : `vault list auth/tokens/accessors`
- Display infos about the token : `vault token lookup -accessor $accessor_id`
- Revoke access : `vault token revoke -accessor $accessor_id`

### Policies
- Capabilities : create, read, update, delete + list, sudo, deny
- Policies are defined using the HashiCorp Configuration Language HCL
- Display capabilities on each path : `vault policy read default`
- Create policy "dev-policy" from file "dev-policy.hcl" : `vault policy write dev-policy dev-policy.hcl`


### Sample use case
```shell script
vault auth enable userpass
vault write auth/userpass/users/app password=app policies=app-policy
vault write auth/userpass/users/dev password=dev policies=dev-policy
vault login -method=userpass password=dev username=dev
vault token capabilities secret/data/dev
vault kv put secret/dev/appsecret user=dbUser
vault login -method=userpass username=app password=app
vault kv get secret/dev/appsecret
vault kv put secret/dev/other key=value
```

## production-style server
1. Set configuration :
    config.hcl :
    ```hcl
        storage "file" {
        path = "/vault/file"
    }
    listener "tcp" {
        address = "0.0.0.0:8200",
        tls_disable = 1
    }
    ```

1. Start server
    ```shell script
    sudo vault server -config=vault/config.hcl
    export VAULT_ADDR=http://0.0.0.0:8200
    vault operator init
    vault operator unseal $unseal_key  # run x3 with 3 diffrerents keys
    vault login $root_token
    vault secrets list
    vault enable ssh
    ```
