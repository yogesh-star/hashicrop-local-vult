# hashicrop-local-vult

export VAULT_TOKEN=s.6zFGB8xNzCZwIjtjkOThg9oe

export VAULT_ADDR="http://13.235.42.5:8200"



cat > values.yaml << EOF
injector:
   enabled: true
   externalVaultAddr: "${VAULT_ADDR}"
EOF


helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update

helm install vault -f values.yaml hashicorp/vault --version "0.10.0"

vault login

vault auth enable kubernetes


export TOKEN_REVIEW_JWT=$(kubectl get secret \
   $(kubectl get serviceaccount vault -o jsonpath='{.secrets[0].name}') \
   -o jsonpath='{ .data.token }' | base64 --decode)

export KUBE_CA_CERT=$(kubectl get secret \
   $(kubectl get serviceaccount vault -o jsonpath='{.secrets[0].name}') \
   -o jsonpath='{ .data.ca\.crt }' | base64 --decode)

export KUBE_HOST=$(kubectl config view --raw --minify --flatten \
   -o jsonpath='{.clusters[].cluster.server}')

vault write auth/kubernetes/config \
   token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
   kubernetes_host="$KUBE_HOST" \
   kubernetes_ca_cert="$KUBE_CA_CERT"

vault write auth/kubernetes/role/app-role \
   bound_service_account_names=app-secret \
   bound_service_account_namespaces=roboot \
   policies=catalogue-policy \
   ttl=10h

cat > catalogue-policy.hcl << EOF
path "secret/catalogue-secret/*" {
  capabilities = ["read"]
}
EOF

vault policy write catalogue-policy catalogue-policy.hcl

vault secrets enable -path=secret/ kv

vault kv put secret/catalogue-secret/catalogue MONGO_URL=mongodb://root:admin12345@docdb-2021-07-13-13-03-21.cnbwakbdzeez.us-east-2.docdb.amazonaws.com:27017/catalogue?ssl=true&ssl_ca_certs=rds-combined-ca-bundle.pem&retryWrites=false

cat /vault/secrets/catalogue
