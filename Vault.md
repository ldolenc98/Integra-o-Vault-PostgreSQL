# **Integrando Vault com o PostgreSQL**

Esta documentação tem como objetivo ensinar a integrar o Vault com o PostgreSQL, gerando credenciais de acesso dinâmicas, garantindo a segurança das informações contidas.

Para iniciarmos nossos estudos devemos antes de mais nada contextualizar as ferramentas utlizadas nesta documentação.

## **PostgreSQL**

Trata-se de uma ferramenta open-source de gerenciamento de banco de dados objeto relacional usada para armazenar informações de forma segura.

## **Vault**

Trata-se de uma ferramenta open-source utilizada para gerenciar o acesso a banco de dados. Com o Vault é possível gerar credênciais dinâmicas, onde cada acesso acontece com um usuário e senha temporários. O uso do Vault é essencial para evitar o vazamento de dados.

Para fazer a integração entre as duas ferramentas é necessário instalar algus pré-requisitos. **Para isso utilize os links**:

* [Instalação do PostgreSQL](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04-pt) ;
* [Instalação do Vault](https://learn.hashicorp.com/tutorials/vault/getting-started-intro?in=vault/getting-started) ;
* [Instalação do Kubectl](https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube).

## **Iniciando a integração**

O primeiro passo para a integração dessas duas ferramentas é clonar o repositório hashicorp/vault-guides, para isso utilizie:

```
$ git clone https://github.com/hashicorp/vault-guides.git
```
Entre no diretório correto utilizando:
```
$ cd vault-guides/operations/provision-vault/kubernetes/minikube/getting-started
```
Crie os namespaces que serão utilizados durante a integração, daremos o nome de "vault" e "postgres".
```
$ kubectl create ns vault
$ kubectl create ns postgres
```
Adicione o repositório HashiCorp Helm.
```
$ helm repo add hashicorp https://helm.releases.hashicorp.com
```
Em seguida, instalaremos o Consul Helm em nosso namespace "vault" aplicando "helm-consul-values.yml" como parâmetro.
```
$ helm install consul hashicorp/consul --values helm-consul-values.yml -n vault
```
Podemos checar a qualquer momento se os pods estão funcionando corretamente utilizando o seguinte comando:
```
$ kubectl get pods -n vault
```
O próximo passo é instalar o Vault Helm em nosso namespace "vault".
```
$ helm install vault hashicorp/vault --values helm-vault-values.yml -n vault
```
Ao checar se os pods estão funcionando corretamente, você deve ter algo parecido com isso:
```
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-server-0                  1/1     Running   0          5m36s
consul-consul-sxpbj                     1/1     Running   0          5m36s
vault-0                                 0/1     Running   0          35s
vault-1                                 0/1     Running   0          34s
vault-2                                 0/1     Running   0          33s
vault-agent-injector-5945fb98b5-wpgx2   1/1     Running   0          36s
```
Nota-se que os pods vault-0, vault-1 e vault-2 não foram inicializados. Este será o nosso próximo passo.

Inicializaremos o Vault com o compartilhamento e limite de chave.
```
$ kubectl exec vault-0 -n vault -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```
Crie uma variável de ambiente que será a nossa chave de inicialização.
```
$ VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```
Utilizaremos esta chave de inicialização para ativar nossos pods, teremos que ativar um de cada vez. Utilize os comandos:
```
$ kubectl exec vault-0 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY

$ kubectl exec vault-1 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY

$ kubectl exec vault-2 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
```
Após ativar nossos pods, você deve ter algo parecido com isso:
```
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-server-0                  1/1     Running   0          10m
consul-consul-sxpbj                     1/1     Running   0          10m
vault-0                                 1/1     Running   0          5m49s
vault-1                                 1/1     Running   0          5m48s
vault-2                                 1/1     Running   0          5m47s
vault-agent-injector-5945fb98b5-vzbqv   1/1     Running   0          5m50s
```

Em seguida, criaremos um pod no namespace "postgres" aplicando o arquivo postgre.yml como parâmetro.
```
$ kubectl apply -f postgre.yml -n postgres
```
Faça login no usuário root do Vault. Para isso, é necessário pegar a root token do arquivo "cluster-keys.json" e utilizar na hora de fazer o login.
```
$ cat cluster-keys.json

$ kubectl exec -ti vault-0 -n vault -- vault login
```
Prepare o ambiente dizendo ao Vault que o secret será um database.
```
$ kubectl exec -ti vault-0 -n vault -- vault secrets enable database
```
Verifique se é possível acessar o postgres. Deve aparecer um terminal próprio do postgres indicando que ainda está sendo possível acessá-lo.
```
$ kubectl exec -it -n postgres $(kubectl get pods -n postgres --selector "app=postgres" -o jsonpath="{.items[0].metadata.name}") -c postgres -- bash -c 'PGPASSWORD=password psql -U postgres'
```

Agora faremos a conexão do Vault com o banco. Para isso pegaremos o IP do namespace e em seguida devemos substituir na segunda linha de código.
```
$ kubectl get services -n postgres

$ kubectl exec -ti vault-0 -n vault -- vault write database/config/dev     plugin_name=postgresql-database-plugin     allowed_roles="*"     connection_url="postgresql://{{username}}:{{password}}@postgres.postgres:5432/dev?sslmode=disable" username="postgres"     password="password"
```
Na linha de código acima, podemos notar um "postgres.postgres". Devemos substituir isso pelo IP do nosso namespace postgres.

Em seguida é necessário declarar uma regra para o acesso ao nosso banco.
```
$ kubectl exec -ti vault-0 -n vault -- vault write database/roles/dev     db_name=dev     creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"     revocation_statements="ALTER ROLE \"{{name}}\" NOLOGIN;"    default_ttl="1h"     max_ttl="24h"
```

Delegue ao Vault o gerenciamento do acesso ao nosso banco. Após esse passo não será mais possível acessar nosso banco da forma casual.
```
$ kubectl exec -ti vault-0 -n vault -- vault write --force /database/rotate-root/dev
```
Agora para conseguirmos acessar nosso banco devemos solicitar ao Vault o acesso. Será gerado um "username" e um "password" cripotagrafados que funcionarão por uma hora. Solicitando o acesso:
```
$ kubectl exec -ti vault-0 -n vault -- vault read database/creds/dev
```
Para acessar o banco, usaremos as credenciais geradas acima e a colocaremos em nossa linha de código.
```
$ kubectl exec -it -n postgres $(kubectl get pods -n postgres --selector "app=postgres" -o jsonpath="{.items[0].metadata.name}") -c postgres -- bash -c 'PGPASSWORD=A1a-3af2yrGLTL3bRoeC psql -U v-root-dev-bhedY39fUUzZHPtmy2Sy-1609169341 dev'
```






























































































































































































































































































