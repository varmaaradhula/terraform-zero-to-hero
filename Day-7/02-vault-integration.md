# Vault Integration

Here are the detailed steps for each of these steps:

## Create an AWS EC2 instance with Ubuntu

To create an AWS EC2 instance with Ubuntu, you can use the AWS Management Console or the AWS CLI. Here are the steps involved in creating an EC2 instance using the AWS Management Console:

- Go to the AWS Management Console and navigate to the EC2 service.
- Click on the Launch Instance button.
- Select the Ubuntu Server xx.xx LTS AMI.
- Select the instance type that you want to use.
- Configure the instance settings.
- Click on the Launch button.

## Install Vault on the EC2 instance

To install Vault on the EC2 instance, you can use the following steps:

**Install gpg**

```
sudo apt update && sudo apt install gpg
```

**Download the signing key to a new keyring**

```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

**Verify the key's fingerprint**

```
gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint
```

**Add the HashiCorp repo**

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

```
sudo apt update
```

**Finally, Install Vault**

```
sudo apt install vault

```

**vault.hcl file**

...
storage "file" {
  path = "/opt/vault/data"
}
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}
ui = true
api_addr = "http://3.10.119.201:8200"
cluster_addr = "http://3.10.119.201:8201"
...

## Start Vault.

To start Vault in dev mode, you can use the following command:

```
vault server -dev -dev-listen-address="0.0.0.0:8200"
```

To start Vault in prod mode, you can use the following command:

```
vault server -config=/etc/vault.d/vault.hcl

sudo nohup vault server -config=/etc/vault.d/vault.hcl > ~/vault.log 2>&1 &

tail -f ~/vault.log

sudo lsof -i :8200


export VAULT_ADDR=http://<public IP>:8200

vault operator init

vault operator unseal  <Enter all 3 unseal keys one by one for each command)

vault login < Need to enter intial token>

vault auth enable approle < need to enable app role before creating role>

```
To run the vault server in background

nohup vault server -dev -dev-listen-address="0.0.0.0:8200" > vault.log 2>&1 &

To check the logs:

tail -f vault.log

## Configure Terraform to read the secret from Vault.

Detailed steps to enable and configure AppRole authentication in HashiCorp Vault:

1. **Enable AppRole Authentication**:

To enable the AppRole authentication method in Vault, you need to use the Vault CLI or the Vault HTTP API.

**Using Vault CLI**:

Run the following command to enable the AppRole authentication method:

```bash
export VAULT_ADDR=http://18.171.156.45:8200
vault login

vault auth enable approle
```

This command tells Vault to enable the AppRole authentication method.

2. **Create an AppRole**:

We need to create policy first,

```
vault policy write terraform - <<EOF
path "*" {
  capabilities = ["list", "read"]
}

path "secrets/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "kv/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}


path "secret/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/token/create" {
capabilities = ["create", "read", "update", "list"]
}
EOF
```

Now you'll need to create an AppRole with appropriate policies and configure its authentication settings. Here are the steps to create an AppRole:

**a. Create the AppRole**:

```bash
vault write auth/approle/role/terraform \
    secret_id_ttl=10m \
    token_num_uses=10 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=40 \
    token_policies=terraform
```

3. **Generate Role ID and Secret ID**:

After creating the AppRole, you need to generate a Role ID and Secret ID pair. The Role ID is a static identifier, while the Secret ID is a dynamic credential.

**a. Generate Role ID**:

vault list auth/approle/role

vault auth list

vault auth enable approle


You can retrieve the Role ID using the Vault CLI:

```bash
vault read auth/approle/role/terraform/role-id
```

Save the Role ID for use in your Terraform configuration.

**b. Generate Secret ID**:

To generate a Secret ID, you can use the following command:

```bash
vault write -f auth/approle/role/terraform/secret-id
   ```

This command generates a Secret ID and provides it in the response. Save the Secret ID securely, as it will be used for Terraform authentication.

**Create secret data**

```bash
vault secrets enable -path=secret/tagnames kv
vault kv put secret/tagnames/tags ubuntu="Assign-Ubuntu" amazon="Assign-RedHat"

vault kv get secret/tagnames/tags
```
**Terraform Integration to get data**
```
***

data "vault_kv_secret_v2" "userdata" {
  mount = "mysecrets" // change it according to your mount
  name  = "userdetails" // change it according to your secret
}

 Name        = data.vault_kv_secret_v2.userdata.data["username"]

***
