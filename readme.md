# requerment 
1. kubernatis or minikube
2. helm chart install 


## Storage: Consul

We will use a very basic Consul cluster for our Vault backend. </br>
Let's find what versions of Consul are available:

```
helm search repo hashicorp/consul --versions

```

mkdir manifests

helm template consul hashicorp/consul  --namespace vault  --version 1.3.0 -f consul-values.yaml  > ./manifests/consul.yaml
```

Deploy the consul services:

```
kubectl create ns vault
kubectl -n vault apply -f ./manifests/consul.yaml
kubectl -n vault get pods
```


## TLS End to End Encryption

# Use CFSSL to generate certificates

```

cd tls

docker run -it --rm -v ${PWD}:/work -w /work debian bash

apt-get update && apt-get install -y curl &&
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64 -o /usr/local/bin/cfssl && \
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64 -o /usr/local/bin/cfssljson && \
chmod +x /usr/local/bin/cfssl && \
chmod +x /usr/local/bin/cfssljson

#generate ca in /tmp
cfssl gencert -initca ca-csr.json | cfssljson -bare /tmp/ca

#generate certificate in /tmp
cfssl gencert \
  -ca=/tmp/ca.pem \
  -ca-key=/tmp/ca-key.pem \
  -config=ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ca-csr.json | cfssljson -bare /tmp/vault
```

view the files:

```
ls -l /tmp
```

access the files:
move the file to tls directory 
```
mv /tmp/* .
exit 
```
Create the TLS secret 

```
kubectl -n vault create secret tls tls-ca \
 --cert ./tls/ca.pem  \
 --key ./tls/ca-key.pem

kubectl -n vault create secret tls tls-server \
  --cert ./tls/vault.pem \
  --key ./tls/vault-key.pem
```

## Generate Kubernetes Manifests


Let's find what versions of vault are available:

```
helm search repo hashicorp/vault --versions
```

In this demo I will use the latest  chart version 0.27.0 (current) </br>

Let's firstly create a `values` file to customize vault.
Let's grab the manifests:

```
helm template vault hashicorp/vault \
  --namespace vault \
  --version 0.27.0 \
  -f vault-values.yaml \
  > ./manifests/vault.yaml
```

## Deployment

```
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
## Web UI

Let's checkout the web UI:

```
kubectl -n vault get svc
kubectl -n vault port-forward svc/vault-ui 443:8200
```

Now we can access the web UI [here](https://localhost/)

## Enable Kubernetes Authentication

For the injector to be authorised to access vault, we need to enable K8s auth

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



# install mysql chart

```
cd mysql
helm install mysql . -n vault
kubectl -n vault exec -it mysql-client -- sh

```
MYSQL - 
    create user hcp identified by "superuser";
    GRANT SELECT, INSERT, UPDATE, DELETE, DROP, CREATE, ALTER, CREATE USER ON *.* TO hcp WITH GRANT OPTION;
    flush privileges;
    

## VAULT 
1. login to vault using cli 

    vault login

2. Enable the database secrets engine 

    vault secrets enable database

3. Configure Vault with the proper plugin and connection information

    vault write database/config/mysql-database \
        plugin_name=mysql-database-plugin \
        connection_url="{{username}}:{{password}}@tcp(mysql)/" \
        allowed_roles="my-role" \
        username="root" \
        password="Chess!125690"

4.Configure a role that maps a name in Vault to an SQL statement to execute to create the database credential:

    vault write database/roles/my-role \
        db_name=mysql-database \
        creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
        default_ttl="1h" \
        max_ttl="24h"


5. Generate a new credential by reading from the /creds endpoint with the name of the role:
    
    vault read database/creds/my-role

