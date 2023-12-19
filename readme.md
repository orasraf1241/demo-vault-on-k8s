# Vault Deployment Guide

## requerment 
1. Kubernetes or Minikube
2. Helm for chart installation


## Storage Setup with Consu

We utilize a basic Consul cluster as our backend for Vault. To determine available Consul versions, use:

```
helm search repo hashicorp/consul --versions

```
Set up the Consul template and deploy:


```
mkdir manifests
helm template consul hashicorp/consul  --namespace vault  --version 1.3.0 -f consul-values.yaml  > ./manifests/consul.yaml

# Deploy the consul services
kubectl create ns vault
kubectl -n vault apply -f ./manifests/consul.yaml
kubectl -n vault get pods
```


## TLS End to End Encryption

# Certificate Generation using CFSSL
1. Navigate to the TLS directory and run a Docker container:

```
cd tls
docker run -it --rm -v ${PWD}:/work -w /work debian bash
```
2. Install CFSSL inside the container:
```
apt-get update && apt-get install -y curl &&
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64 -o /usr/local/bin/cfssl && \
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64 -o /usr/local/bin/cfssljson && \
chmod +x /usr/local/bin/cfssl && \
chmod +x /usr/local/bin/cfssljson
```

3. Generate certificates:

```
# Generate CA in /tmp
cfssl gencert -initca ca-csr.json | cfssljson -bare /tmp/ca

# Generate vault certificate in /tmp
cfssl gencert \
  -ca=/tmp/ca.pem \
  -ca-key=/tmp/ca-key.pem \
  -config=ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ca-csr.json | cfssljson -bare /tmp/vault
```

4. Move the generated files to the TLS directory:

```
mv /tmp/* .
exit
```

5. Create TLS secrets in Kubernetes:


```
kubectl -n vault create secret tls tls-ca \
 --cert ./tls/ca.pem  \
 --key ./tls/ca-key.pem

kubectl -n vault create secret tls tls-server \
  --cert ./tls/vault.pem \
  --key ./tls/vault-key.pem
```

## Generate Kubernetes Manifests for Vault

Check for available Vault versions:

```
helm search repo hashicorp/vault --versions
```
For this demo, we use chart version 0.27.0:

1. Create a 'values' file for customization.
2. Generate and apply Vault manifests:

## Initializing Vault
Execute these commands:

```
helm template vault hashicorp/vault --namespace vault --version 0.27.0 -f vault-values.yaml > ./manifests/vault.yaml
kubectl -n vault apply -f ./manifests/vault.yaml
kubectl -n vault get pods
```



## Initialising Vault

```
kubectl -n vault exec -it vault-0 -- sh
kubectl -n vault exec -it vault-1 -- sh
kubectl -n vault exec -it vault-2 -- sh

vault operator init
vault operator unseal

kubectl -n vault exec -it vault-0 -- vault status
kubectl -n vault exec -it vault-1 -- vault status
kubectl -n vault exec -it vault-2 -- vault status

```
## Accessing the Web UI


To access the Vault Web UI:

```
kubectl -n vault get svc
kubectl -n vault port-forward svc/vault-ui 443:8200
```

Access the UI at https://localhost/


## Enable Kubernetes Authentication

To authorize the injector for Vault access:


```
kubectl -n vault exec -it vault-0 -- sh 

vault login
vault auth enable kubernetes

vault write auth/kubernetes/config \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
issuer="https://kubernetes.default.svc.cluster.local"
```



# Install MySQL Chart
Setup MySQL within the Vault namespace:


```
cd mysql
helm install mysql . -n vault
kubectl -n vault exec -it mysql-client -- sh
```

Configure MySQL user:
```
    mysql -u root -h mysql -p
    create user hcp identified by "superuser";
    GRANT SELECT, INSERT, UPDATE, DELETE, DROP, CREATE, ALTER, CREATE USER ON *.* TO hcp WITH GRANT OPTION;
    flush privileges;
``` 

## Vault Database Secrets Engine 
1. Log in to Vault CLI:
```
    kubectl -n vault exec -it vault-0 -- sh
    vault login

```
2. Enable the database secrets engine 
```
    vault secrets enable database
```
3. Configure Vault with MySQL plugin and connection details:
```
vault write database/config/mysql-database \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql)/" \
    allowed_roles="my-role" \
    username="hcp" \
    password="PASSWORD"
```
4.Create a role for database credentials:
```
    vault write database/roles/my-role \
        db_name=mysql-database \
        creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
        default_ttl="1h" \
        max_ttl="24h"
```

5. Generate username and pass:
    ```
    vault read database/creds/my-role
```













