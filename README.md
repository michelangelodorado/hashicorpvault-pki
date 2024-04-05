# hashicorpvault-pki

Guide to:

1. Deploy a local development Vault Server
2. Configure Vault for PKI certificate management (self-signed)
3. Use Vault Agent to write certificates to a file for applications/NGINX to use.

Note: This are all tested using macOS


```shell
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

```shell
vault server -dev -dev-root-token-id root
```

```shell
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root
```

```shell
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
```

```shell
vault write -field=certificate pki/root/generate/internal \
     common_name="example.com" \
     issuer_name="root-2024" \
     ttl=87600h > root_2024_ca.crt
```

```shell
vault list pki/issuers/
```

```shell
vault read pki/issuer/$(vault list -format=json pki/issuers/ | jq -r '.[]') \
 | tail -n 6
```

```shell
vault write pki/roles/2023-servers allow_any_name=true
```

```shell
vault write pki/config/urls \
     issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
     crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

```shell
vault secrets enable -path=pki_int pki
```

```shell
vault secrets tune -max-lease-ttl=43800h pki_int
```

```shell
vault write -format=json pki_int/intermediate/generate/internal \
     common_name="example.com Intermediate Authority" \
     issuer_name="example-dot-com-intermediate" \
     | jq -r '.data.csr' > pki_intermediate.csr
```

```shell
vault write -format=json pki/root/sign-intermediate \
     issuer_ref="root-2024" \
     csr=@pki_intermediate.csr \
     format=pem_bundle ttl="43800h" \
     | jq -r '.data.certificate' > intermediate.cert.pem
```

```shell
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```

```shell
vault write pki_int/roles/example-dot-com \
     issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
     allowed_domains="example.com" \
     allow_subdomains=true \
     max_ttl="720h"
```

---

```shell
echo 'path "pki_int/*" {
  capabilities = ["read","create","update"]
}' | vault policy write certs -
```

```shell
vault auth enable approle
```

```shell
vault write auth/approle/role/example-dot-com \
   role_id=example-dot-com \
   secret_id_ttl=30m \
   token_num_uses=0 \
   token_ttl=30m \
   token_max_ttl=60m \
   token_policies=certs \
   secret_id_num_uses=0
```

```shell
vault read -field=role_id \
   auth/approle/role/example-dot-com/role-id > vault_agent_role_id
```

```shell
vault write -f -field=secret_id \
   auth/approle/role/example-dot-com/secret-id > vault_agent_secret_id
```

```shell
mkdir templates
```

```shell
cat << EOF > templates/ca.tpl
{{ with pkiCert "pki_int/issue/example-dot-com" "common_name=test.example.com" }}{{ .CA }}{{ end }}
EOF
```

```shell
cat << EOF > templates/cert.tpl
{{ with pkiCert "pki_int/issue/example-dot-com" "common_name=test.example.com" }}{{ .Cert }}{{ end }}
EOF
```

```shell
cat << EOF > templates/key.tpl
{{ with pkiCert "pki_int/issue/example-dot-com" "common_name=test.example.com" }}{{ .Key }}{{ end }}
EOF
```

```shell
cat << EOF > vault-agent.hcl
pid_file        = "./pidfile"
exit_after_auth = true
vault {
  address = "http://127.0.0.1:8200"
}
auto_auth {
  method {
    type = "approle"
    config = {
      role_id_file_path                   = "vault_agent_role_id"
      secret_id_file_path                 = "vault_agent_secret_id"
      remove_secret_id_file_after_reading = false
    }
  }
  sink {
    type = "file"
    config = {
      path = "vault_agent_token"
    }
  }
}
template {
  source      = "templates/cert.tpl"
  destination = "examples/my-app.crt"
}
template {
  source      = "templates/ca.tpl"
  destination = "examples/ca.crt"
}
template {
  source      = "templates/key.tpl"
  destination = "examples/my-app.key"
}
EOF
```

```shell
vault agent -config=vault-agent.hcl
```

