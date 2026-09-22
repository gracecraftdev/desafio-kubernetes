# Desafio — Fundamentos de Kubernetes na Prática

Este repositório contém a implementação do desafio prático de fundamentos de Kubernetes.

O objetivo do projeto foi criar uma aplicação utilizando **PostgREST** integrada a um banco de dados **PostgreSQL**, executando os componentes em um cluster Kubernetes local.

Durante o desafio foram utilizados recursos como:

- Namespace;
- Pods;
- Deployments;
- Services;
- ConfigMaps;
- Secrets;
- PersistentVolumeClaims;
- Liveness Probe;
- Readiness Probe;
- Requests e Limits;
- escalabilidade horizontal;
- Horizontal Pod Autoscaler (HPA);
- Metrics Server.

---

# Arquitetura

A arquitetura da aplicação é composta por:

```text
Usuário
   |
   | HTTP
   v
PostgREST Service
   |
   v
PostgREST Deployment
   |
   | SQL
   v
PostgreSQL Service
   |
   v
PostgreSQL Deployment
   |
   v
PersistentVolumeClaim
```

O PostgREST se comunica com o PostgreSQL utilizando o **nome do Service `postgres`**, e não o endereço IP do Pod.

Dessa forma, a aplicação não depende de IPs individuais, que podem mudar quando Pods são recriados.

---

# Tecnologias utilizadas

- Kubernetes
- Docker Desktop
- kubectl
- PostgreSQL 16
- PostgREST
- Metrics Server
- Horizontal Pod Autoscaler
- Git
- GitHub
- WSL2
- Ubuntu

---

# Estrutura do projeto

```text
desafio-kubernetes/
├── k8s/
│   ├── 01-namespace.yaml
│   ├── 02-pod-teste.yaml
│   ├── 03-postgres-pvc.yaml
│   ├── 03-secret.example.yaml
│   ├── 04-configmap.yaml
│   ├── 05-postgres-deployment.yaml
│   ├── 06-postgres-service.yaml
│   ├── 07-postgrest-deployment.yaml
│   ├── 08-postgrest-service.yaml
│   └── 09-postgrest-hpa.yaml
│
├── evidencias/
│   ├── 01-nivel-1-cluster-namespace-pod.png
│   ├── 02-nivel-1-inspecao-pod.png
│   ├── 03-nivel-1-exclusao-pod.png
│   ├── 04-niveis-2-3-postgresql-configuracao.png
│   ├── 05-nivel-4-api-postgrest.png
│   ├── 06-nivel-5-persistencia-pvc.png
│   ├── 07-nivel-6-health-recursos-scaling.png
│   ├── 08-estado-final-kubernetes.png
│   └── 09-nivel-7-hpa.png
│
├── .gitignore
└── README.md
```

---

# Nível 1 — Namespace e Pod de teste

Foi criado um Namespace exclusivo para o desafio:

```bash
kubectl apply -f k8s/01-namespace.yaml
```

Para confirmar sua criação:

```bash
kubectl get namespaces
```

Em seguida, foi criado um Pod de teste utilizando a imagem:

```text
nginx:alpine
```

Aplicação:

```bash
kubectl apply -f k8s/02-pod-teste.yaml
```

Verificação:

```bash
kubectl get pods -n desafio-kubernetes -o wide
```

Também foram utilizados comandos para inspecionar o Pod:

```bash
kubectl describe pod pod-teste -n desafio-kubernetes
```

e visualizar seus logs:

```bash
kubectl logs pod-teste -n desafio-kubernetes
```

Depois, o Pod foi excluído:

```bash
kubectl delete pod pod-teste -n desafio-kubernetes
```

Como o Pod havia sido criado diretamente, sem ser gerenciado por um Deployment, ReplicaSet ou outro controlador, ele **não foi recriado automaticamente**.

## Evidências

![Cluster, Namespace e Pod](evidencias/01-nivel-1-cluster-namespace-pod.png)

![Inspeção do Pod](evidencias/02-nivel-1-inspecao-pod.png)

![Exclusão do Pod](evidencias/03-nivel-1-exclusao-pod.png)

---

# Nível 2 — PostgreSQL e persistência

Foi criado um PersistentVolumeClaim para armazenar os dados do PostgreSQL.

Arquivo:

```text
k8s/03-postgres-pvc.yaml
```

O PVC solicita:

```text
1Gi
```

de armazenamento utilizando o modo:

```text
ReadWriteOnce
```

O PostgreSQL utiliza a imagem:

```text
postgres:16
```

O volume persistente é montado em:

```text
/var/lib/postgresql/data
```

Também foi configurada a variável:

```text
PGDATA=/var/lib/postgresql/data/pgdata
```

para utilização do diretório persistente.

O PostgreSQL é disponibilizado dentro do cluster por um Service do tipo:

```text
ClusterIP
```

na porta:

```text
5432
```

## PVC x emptyDir

Um `emptyDir` existe apenas durante o ciclo de vida do Pod.

Se o Pod for removido, os dados armazenados nele também são perdidos.

Já um `PersistentVolumeClaim` possui um ciclo de vida independente do Pod. Dessa forma, quando o PostgreSQL é recriado, o novo Pod pode montar novamente o mesmo volume e acessar os dados existentes.

---

# Nível 3 — ConfigMap e Secret

As configurações comuns da aplicação foram armazenadas em um ConfigMap.

Arquivo:

```text
k8s/04-configmap.yaml
```

Entre as configurações utilizadas estão:

```text
POSTGRES_DB
PGRST_DB_SCHEMAS
PGRST_DB_ANON_ROLE
```

As credenciais do banco foram armazenadas em um Secret do Kubernetes e não foram adicionadas diretamente aos Deployments.

O repositório contém apenas um exemplo:

```text
k8s/03-secret.example.yaml
```

Exemplo:

```yaml
apiVersion: v1
kind: Secret

metadata:
  name: postgres-secret
  namespace: desafio-kubernetes

type: Opaque

stringData:
  POSTGRES_USER: desafio_user
  POSTGRES_PASSWORD: SUA_SENHA_FORTE
  PGRST_DB_URI: postgres://desafio_user:SUA_SENHA_URL_ENCODED@postgres:5432/desafio_db
```

O arquivo contendo credenciais reais não deve ser versionado.

Por esse motivo, arquivos de Secret reais são ignorados pelo `.gitignore`.

## URL encoding

Caso a senha contenha caracteres reservados de uma URI, como `@`, `%`, `:` ou `/`, é necessário utilizar URL encoding antes de inserir a senha na string de conexão.

Exemplo da estrutura:

```text
postgres://desafio_user:SUA_SENHA_URL_ENCODED@postgres:5432/desafio_db
```

## Base64 não é criptografia

Os valores armazenados no campo `data` de um Secret do Kubernetes utilizam Base64.

Base64 é apenas uma codificação e **não representa criptografia**.

Por isso, Secrets devem continuar sendo tratados como informações sensíveis e não devem ser publicados em repositórios.

## Evidência

![PostgreSQL, PVC, ConfigMap e Secret](evidencias/04-niveis-2-3-postgresql-configuracao.png)

---

# Nível 4 — Integração PostgREST e PostgreSQL

Foi criado um Deployment para executar o PostgREST.

Arquivo:

```text
k8s/07-postgrest-deployment.yaml
```

O PostgREST recebe sua string de conexão através do Secret:

```text
PGRST_DB_URI
```

A conexão utiliza:

```text
postgres
```

como hostname.

Esse nome corresponde ao Service do PostgreSQL dentro do Kubernetes.

Assim, a aplicação utiliza o DNS interno do cluster em vez de depender diretamente do IP de um Pod.

---

## Estrutura criada no PostgreSQL

Foi criada uma role para acesso da API:

```sql
CREATE ROLE web_anon NOLOGIN;
```

A role foi associada ao usuário utilizado pela aplicação:

```sql
GRANT web_anon TO desafio_user;
```

Também foi criada a tabela:

```sql
CREATE TABLE public.items (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
```

As permissões necessárias foram concedidas:

```sql
GRANT USAGE ON SCHEMA public TO web_anon;

GRANT SELECT, INSERT, UPDATE, DELETE
ON public.items
TO web_anon;

GRANT USAGE, SELECT
ON SEQUENCE public.items_id_seq
TO web_anon;
```

---

# Acesso à API

O PostgREST é disponibilizado internamente por um Service do tipo `ClusterIP`.

Para acessar a API a partir da máquina local, foi utilizado:

```bash
kubectl port-forward -n desafio-kubernetes svc/postgrest 3000:3000
```

A API fica disponível localmente na porta:

```text
3000
```

---

# Testes da API

## Inserção de dados

Foi realizada uma requisição `POST` para inserir um registro:

```bash
curl -i -X POST http://127.0.0.1:3000/items \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"name":"dado-persistente"}'
```

A API retornou:

```text
HTTP/1.1 201 Created
```

com o registro criado.

## Consulta

Para consultar os registros:

```bash
curl -i http://127.0.0.1:3000/items
```

O registro inserido foi retornado pela API.

## Evidência

![API PostgREST](evidencias/05-nivel-4-api-postgrest.png)

---

# Nível 5 — Teste de persistência

Um dos principais testes do projeto foi verificar se os dados permaneceriam disponíveis mesmo após a exclusão do Pod do PostgreSQL.

Primeiro, o registro foi consultado pela API:

```json
[
  {
    "id": 1,
    "name": "dado-persistente"
  }
]
```

Em seguida, o Pod do PostgreSQL foi removido.

Como o PostgreSQL é gerenciado por um Deployment, o Kubernetes detectou que a quantidade desejada de réplicas não estava sendo atendida e criou automaticamente um novo Pod.

Depois que o novo Pod ficou disponível, a API foi consultada novamente.

O mesmo registro continuava armazenado.

Isso demonstrou que os dados não estavam associados ao ciclo de vida do Pod, mas ao PersistentVolumeClaim.

## Evidência

![Persistência com PVC](evidencias/06-nivel-5-persistencia-pvc.png)

---

# Nível 6 — Probes, recursos e escalabilidade

O Deployment do PostgREST foi configurado inicialmente com:

```yaml
replicas: 2
```

Também foram adicionadas verificações de saúde.

## Readiness Probe

A Readiness Probe verifica se o container está pronto para receber tráfego.

Foi utilizada uma requisição HTTP para:

```text
/
```

na porta:

```text
3000
```

Se o Pod não estiver pronto, o Kubernetes deixa de encaminhar novas requisições para ele através do Service.

## Liveness Probe

A Liveness Probe verifica se o container continua funcionando.

Foi utilizada uma verificação TCP na porta:

```text
3000
```

Caso a verificação falhe repetidamente, o Kubernetes pode reiniciar o container.

## Requests e Limits

Também foram configurados recursos de CPU e memória:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

Os `requests` indicam os recursos necessários para o agendamento do Pod.

Os `limits` estabelecem o máximo de recursos que o container pode utilizar.

A definição de `requests.cpu` também permite que o HPA calcule posteriormente a utilização percentual de CPU.

## Escalabilidade

Com duas réplicas do PostgREST, o Service possui múltiplos endpoints disponíveis.

Isso permite distribuir as requisições entre diferentes Pods.

A API pode ser escalada horizontalmente porque é um componente stateless.

Já o PostgreSQL utiliza armazenamento persistente com PVC `ReadWriteOnce`, portanto aumentar suas réplicas da mesma forma exigiria uma estratégia própria para banco de dados, replicação e armazenamento compartilhado.

## Evidência

![Health Checks, recursos e scaling](evidencias/07-nivel-6-health-recursos-scaling.png)

---

# Nível 7 — Horizontal Pod Autoscaler (HPA)

Como etapa bônus, foi configurado o escalonamento horizontal automático do PostgREST com base no consumo de CPU.

## Metrics Server

O HPA precisa de métricas para tomar decisões de escalabilidade.

Para isso, foi instalado o Metrics Server no cluster.

Após a instalação inicial, o Metrics Server apresentou um erro relacionado à validação do certificado TLS do kubelet no ambiente Kubernetes local.

O erro indicava que o certificado não continha o IP do node nos SANs.

Como se trata de um ambiente local de laboratório, foi adicionada ao Metrics Server a opção:

```text
--kubelet-insecure-tls
```

Após a alteração, o Pod do Metrics Server ficou:

```text
1/1 Running
```

e as métricas passaram a ficar disponíveis.

A coleta foi validada utilizando:

```bash
kubectl top nodes
kubectl top pods -n desafio-kubernetes
```

---

## Configuração do HPA

O HPA está definido no arquivo:

```text
k8s/09-postgrest-hpa.yaml
```

A configuração utiliza:

```yaml
minReplicas: 2
maxReplicas: 5
```

e possui como alvo:

```yaml
averageUtilization: 50
```

Isso significa que o Kubernetes monitora a utilização média de CPU do Deployment do PostgREST e pode ajustar automaticamente a quantidade de réplicas entre 2 e 5.

O HPA foi aplicado com:

```bash
kubectl apply -f k8s/09-postgrest-hpa.yaml
```

Inicialmente, com a aplicação sem carga significativa, foi observado:

```text
cpu: 4%/50%
```

com:

```text
2 réplicas
```

---

## Teste de carga

Para testar o HPA, foi criado temporariamente um Pod chamado `load-generator`.

Ele realizou múltiplas requisições continuamente para:

```text
http://postgrest:3000/items
```

Durante o teste, o consumo de CPU aumentou significativamente.

O HPA registrou:

```text
cpu: 375%/50%
```

e aumentou automaticamente o número de réplicas do PostgREST.

O Deployment passou de:

```text
2 réplicas
```

para:

```text
5 réplicas
```

O resultado observado foi:

```text
READY   UP-TO-DATE   AVAILABLE
5/5     5            5
```

e os cinco Pods estavam em estado:

```text
1/1 Running
```

Isso confirmou o funcionamento do **Horizontal Pod Autoscaler baseado em CPU**.

Após o teste, o Pod utilizado para gerar a carga foi removido:

```bash
kubectl delete pod load-generator -n desafio-kubernetes
```

## Evidência

![Horizontal Pod Autoscaler](evidencias/09-nivel-7-hpa.png)

---

# Estado final do cluster

Antes do teste bônus de HPA, foi registrado o estado dos principais recursos do projeto utilizando:

```bash
kubectl get all -n desafio-kubernetes
```

O resultado confirmou:

- PostgreSQL com `1/1` Pod em execução;
- PostgREST com `2/2` réplicas disponíveis;
- Services `postgres` e `postgrest`;
- Deployments disponíveis;
- ReplicaSets responsáveis pelo gerenciamento dos Pods.

![Estado final do Kubernetes](evidencias/08-estado-final-kubernetes.png)

Posteriormente, durante o teste do Nível 7, o HPA aumentou temporariamente o PostgREST para 5 réplicas em resposta à carga gerada.

---

# Ordem de aplicação dos manifests

Os principais manifests podem ser aplicados na seguinte ordem:

```bash
kubectl apply -f k8s/01-namespace.yaml

kubectl apply -f k8s/03-postgres-pvc.yaml

kubectl apply -f k8s/04-configmap.yaml

kubectl apply -f k8s/05-postgres-deployment.yaml

kubectl apply -f k8s/06-postgres-service.yaml

kubectl apply -f k8s/07-postgrest-deployment.yaml

kubectl apply -f k8s/08-postgrest-service.yaml

kubectl apply -f k8s/09-postgrest-hpa.yaml
```

O Secret com as credenciais reais deve ser criado separadamente antes dos Deployments que dependem dele.

O arquivo `03-secret.example.yaml` serve apenas como referência e não contém credenciais reais.

---

# Verificação dos recursos

Alguns comandos úteis utilizados durante o desafio:

```bash
kubectl get all -n desafio-kubernetes

kubectl get pods -n desafio-kubernetes

kubectl get services -n desafio-kubernetes

kubectl get pvc -n desafio-kubernetes

kubectl get hpa -n desafio-kubernetes

kubectl top pods -n desafio-kubernetes
```

Para visualizar detalhes de um recurso:

```bash
kubectl describe pod <nome-do-pod> -n desafio-kubernetes
```

Para visualizar logs:

```bash
kubectl logs <nome-do-pod> -n desafio-kubernetes
```

---

# Segurança

As credenciais reais utilizadas durante o desafio não são armazenadas no repositório.

O `.gitignore` impede o versionamento de arquivos locais contendo Secrets.

O repositório disponibiliza apenas:

```text
k8s/03-secret.example.yaml
```

com valores fictícios.

Antes de realizar commits, é importante verificar se nenhuma credencial foi adicionada acidentalmente ao projeto.

---

# Versionamento com Git

O projeto foi versionado em etapas lógicas durante o desenvolvimento.

Exemplos de etapas versionadas:

- criação do Namespace e Pod de teste;
- configuração do PostgreSQL e armazenamento persistente;
- integração com PostgREST;
- teste de persistência;
- configuração de health checks e recursos;
- escalabilidade da API;
- configuração do HPA;
- documentação e evidências.

Comandos utilizados:

```bash
git status
git add .
git commit -m "mensagem do commit"
git push
```

---

# Principais conceitos praticados

Durante o desenvolvimento deste desafio foram praticados:

- Namespace;
- Pods;
- Deployments;
- ReplicaSets;
- Services;
- ClusterIP;
- DNS interno do Kubernetes;
- ConfigMaps;
- Secrets;
- PersistentVolumeClaims;
- armazenamento persistente;
- ciclo de vida de Pods;
- integração entre aplicações;
- PostgREST;
- PostgreSQL;
- Liveness Probe;
- Readiness Probe;
- Requests e Limits;
- escalabilidade horizontal;
- Horizontal Pod Autoscaler (HPA);
- Metrics Server;
- métricas de CPU;
- geração de carga;
- EndpointSlices;
- troubleshooting;
- versionamento com Git.

---

# Conclusão

O desafio permitiu aplicar na prática os principais fundamentos do Kubernetes.

A aplicação foi dividida em componentes independentes, utilizando Services para comunicação interna e evitando dependência direta dos IPs dos Pods.

O PostgreSQL utiliza armazenamento persistente por meio de um PVC, permitindo que os dados continuem disponíveis mesmo após a exclusão e recriação do Pod.

As credenciais foram separadas das configurações comuns utilizando Secret e ConfigMap.

O PostgREST foi configurado com health checks, gerenciamento de recursos e múltiplas réplicas, permitindo observar conceitos relacionados à disponibilidade e escalabilidade.

Também foi configurado um Horizontal Pod Autoscaler baseado em CPU. Durante o teste de carga, o HPA aumentou automaticamente o PostgREST de 2 para 5 réplicas, demonstrando o escalonamento horizontal da aplicação em resposta ao aumento de utilização.

Além da criação dos recursos, foram realizados testes de integração, comunicação entre os componentes, exclusão e recriação de Pods, persistência de dados, health checks, coleta de métricas e escalabilidade automática.

Com isso, o projeto demonstra na prática o funcionamento dos principais recursos utilizados para executar e gerenciar uma aplicação no Kubernetes.