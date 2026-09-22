# Desafio — Fundamentos de Kubernetes na Prática

Este repositório reúne o que desenvolvi durante o desafio prático de Kubernetes. A proposta foi subir uma API com **PostgREST** conectada a um banco **PostgreSQL**, rodando tudo em um cluster Kubernetes local.

Ao longo do desafio fui adicionando os recursos por etapas: comecei com Namespace e Pod, depois configurei persistência, comunicação entre os serviços, health checks, limites de recursos e, por último, o HPA para testar o escalonamento automático da API.

## Tecnologias utilizadas

- Kubernetes
- Docker Desktop
- kubectl
- PostgreSQL 16
- PostgREST
- Metrics Server
- HPA (Horizontal Pod Autoscaler)
- Git e GitHub
- WSL2 / Ubuntu

## Arquitetura

A estrutura da aplicação ficou assim:

```mermaid
flowchart TD
    A[Usuário] -->|HTTP| B[PostgREST Service]
    B --> C[PostgREST Deployment]
    C -->|SQL| D[PostgreSQL Service]
    D --> E[PostgreSQL Deployment]
    E --> F[PersistentVolumeClaim]
```

A comunicação entre o PostgREST e o PostgreSQL é feita pelo Service `postgres`. Dessa forma, não preciso depender diretamente do IP de um Pod, que pode mudar quando ele é recriado.

## Estrutura do projeto

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

## Nível 1 — Namespace e Pod de teste

Comecei criando um Namespace separado para o desafio:

```bash
kubectl apply -f k8s/01-namespace.yaml
kubectl get namespaces
```

Depois subi um Pod de teste usando `nginx:alpine`:

```bash
kubectl apply -f k8s/02-pod-teste.yaml
kubectl get pods -n desafio-kubernetes -o wide
```

Para conferir melhor o que estava acontecendo com o Pod, usei:

```bash
kubectl describe pod pod-teste -n desafio-kubernetes
kubectl logs pod-teste -n desafio-kubernetes
```

Depois do teste, excluí o Pod:

```bash
kubectl delete pod pod-teste -n desafio-kubernetes
```

Como ele tinha sido criado diretamente, sem Deployment ou outro controlador, o Kubernetes não criou outro Pod no lugar. Esse teste ajudou a visualizar bem a diferença entre um Pod isolado e um Pod gerenciado.

### Evidências

![Cluster, Namespace e Pod](evidencias/01-nivel-1-cluster-namespace-pod.png)

![Inspeção do Pod](evidencias/02-nivel-1-inspecao-pod.png)

![Exclusão do Pod](evidencias/03-nivel-1-exclusao-pod.png)

---

## Nível 2 — PostgreSQL e persistência

Na segunda etapa configurei o PostgreSQL e o armazenamento persistente.

O PVC está no arquivo:

```text
k8s/03-postgres-pvc.yaml
```

Ele solicita `1Gi` de armazenamento no modo `ReadWriteOnce`.

O PostgreSQL usa a imagem:

```text
postgres:16
```

O volume é montado em:

```text
/var/lib/postgresql/data
```

e configurei:

```text
PGDATA=/var/lib/postgresql/data/pgdata
```

O banco fica disponível dentro do cluster por um Service `ClusterIP` na porta `5432`.

### Por que usei PVC?

Um `emptyDir` acompanha o ciclo de vida do Pod. Se o Pod for removido, os dados também são perdidos.

Com o `PersistentVolumeClaim`, o armazenamento fica separado do ciclo de vida do Pod. Assim, se o PostgreSQL for recriado, o novo Pod consegue montar o volume novamente e continuar usando os dados que já estavam salvos.

---

## Nível 3 — ConfigMap e Secret

Separei as configurações comuns das informações sensíveis.

No ConfigMap ficaram valores como:

```text
POSTGRES_DB
PGRST_DB_SCHEMAS
PGRST_DB_ANON_ROLE
```

As credenciais ficaram em um Secret. O repositório possui somente um arquivo de exemplo:

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

O Secret real não foi versionado e está protegido pelo `.gitignore`.

Um detalhe importante nessa etapa foi a senha usada na URI de conexão.

Se ela tiver caracteres reservados, como `@`, `%`, `:` ou `/`, é necessário aplicar URL encoding.

Também vale lembrar que Base64 não é criptografia. Mesmo quando um Secret usa valores codificados em Base64, ele continua sendo informação sensível e não deve ser publicado no repositório.

### Evidência

![PostgreSQL, PVC, ConfigMap e Secret](evidencias/04-niveis-2-3-postgresql-configuracao.png)

---

## Nível 4 — PostgREST + PostgreSQL

Depois do banco funcionando, configurei o Deployment do PostgREST:

```text
k8s/07-postgrest-deployment.yaml
```

A string de conexão é recebida pelo `PGRST_DB_URI`, que vem do Secret.

Na URI, o hostname utilizado é `postgres`, que corresponde ao Service do PostgreSQL. Com isso, a comunicação usa o DNS interno do Kubernetes em vez do IP de um Pod.

### Configuração feita no PostgreSQL

Criei uma role para o acesso da API:

```sql
CREATE ROLE web_anon NOLOGIN;
GRANT web_anon TO desafio_user;
```

Depois criei uma tabela simples para os testes:

```sql
CREATE TABLE public.items (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
```

E concedi as permissões necessárias:

```sql
GRANT USAGE ON SCHEMA public TO web_anon;

GRANT SELECT, INSERT, UPDATE, DELETE
ON public.items
TO web_anon;

GRANT USAGE, SELECT
ON SEQUENCE public.items_id_seq
TO web_anon;
```

### Acessando a API

O PostgREST também fica atrás de um Service `ClusterIP`. Para acessar a API pela minha máquina, usei port-forward:

```bash
kubectl port-forward -n desafio-kubernetes svc/postgrest 3000:3000
```

Com isso, a API ficou disponível localmente na porta `3000`.

### Testando a API

Para inserir um registro:

```bash
curl -i -X POST http://127.0.0.1:3000/items \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"name":"dado-persistente"}'
```

A resposta foi:

```text
HTTP/1.1 201 Created
```

Depois consultei os dados:

```bash
curl -i http://127.0.0.1:3000/items
```

O registro criado apareceu normalmente na resposta.

### Evidência

![API PostgREST](evidencias/05-nivel-4-api-postgrest.png)

---

## Nível 5 — Teste de persistência

Aqui fiz um dos testes mais importantes do desafio.

Primeiro confirmei que o registro estava salvo:

```json
[
  {
    "id": 1,
    "name": "dado-persistente"
  }
]
```

Depois excluí o Pod do PostgreSQL.

Como o banco está sendo gerenciado por um Deployment, o Kubernetes percebeu que faltava uma réplica e criou outro Pod automaticamente.

Quando o novo Pod ficou pronto, consultei a API de novo e o registro continuava lá.

Na prática, esse teste mostrou que os dados estavam no volume persistente e não presos ao Pod que eu havia excluído.

### Evidência

![Persistência com PVC](evidencias/06-nivel-5-persistencia-pvc.png)

---

## Nível 6 — Health checks, recursos e escalabilidade

Nessa etapa deixei o PostgREST com duas réplicas:

```yaml
replicas: 2
```

Também configurei Readiness Probe e Liveness Probe.

### Readiness Probe

A Readiness Probe verifica se o container já está pronto para receber requisições. Usei uma checagem HTTP no endpoint `/`, porta `3000`.

Enquanto um Pod não estiver pronto, ele não deve receber novas requisições pelo Service.

### Liveness Probe

Na Liveness Probe usei uma verificação TCP na porta `3000`.

Ela ajuda o Kubernetes a identificar quando o container deixou de responder corretamente e precisa ser reiniciado.

### Requests e Limits

Configurei CPU e memória:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

Os `requests` representam os recursos considerados no agendamento do Pod, enquanto os `limits` definem o limite de uso do container.

O `requests.cpu` também foi importante para o HPA, já que a porcentagem de utilização de CPU é calculada usando esse valor como referência.

### Escalabilidade

Com duas réplicas do PostgREST, o Service consegue encaminhar requisições para mais de um Pod.

Nesse caso faz sentido escalar a API horizontalmente porque ela é stateless.

Para o PostgreSQL a situação é diferente. Como ele utiliza armazenamento persistente com PVC `ReadWriteOnce`, simplesmente aumentar o número de réplicas não seria suficiente. Seria necessário pensar também em replicação do banco e na estratégia de armazenamento.

### Evidência

![Health Checks, recursos e scaling](evidencias/07-nivel-6-health-recursos-scaling.png)

---

## Nível 7 — Horizontal Pod Autoscaler

Como etapa bônus, configurei o HPA para alterar automaticamente a quantidade de Pods do PostgREST de acordo com o consumo de CPU.

### Metrics Server

Para o HPA funcionar, primeiro precisei instalar o Metrics Server.

Foi nessa parte que encontrei um problema: depois da instalação, o Metrics Server não conseguia validar o certificado TLS do kubelet no meu ambiente local. O erro indicava que o certificado não possuía o IP do node nos SANs.

Como era um cluster local de laboratório, ajustei o Metrics Server usando:

```text
--kubelet-insecure-tls
```

Depois da alteração, o Pod ficou `1/1 Running` e consegui consultar as métricas:

```bash
kubectl top nodes
kubectl top pods -n desafio-kubernetes
```

Esse foi um dos pontos do desafio em que precisei fazer troubleshooting em vez de apenas aplicar os manifests.

### Configuração do HPA

O HPA está em:

```text
k8s/09-postgrest-hpa.yaml
```

A configuração permite entre 2 e 5 réplicas:

```yaml
minReplicas: 2
maxReplicas: 5
```

com alvo médio de CPU de:

```yaml
averageUtilization: 50
```

Apliquei com:

```bash
kubectl apply -f k8s/09-postgrest-hpa.yaml
```

Sem carga significativa, o HPA mostrava aproximadamente:

```text
cpu: 4%/50%
```

e o Deployment permanecia com 2 réplicas.

### Teste de carga

Para ver o HPA funcionando de verdade, criei temporariamente um Pod `load-generator` fazendo várias requisições para:

```text
http://postgrest:3000/items
```

Durante o teste, o HPA chegou a registrar:

```text
cpu: 375%/50%
```

Com o aumento de CPU, o Kubernetes escalou o PostgREST de **2 para 5 réplicas**.

O Deployment chegou ao estado:

```text
READY   UP-TO-DATE   AVAILABLE
5/5     5            5
```

e os cinco Pods ficaram `1/1 Running`.

Depois do teste removi o gerador de carga:

```bash
kubectl delete pod load-generator -n desafio-kubernetes
```

### Evidência

![Horizontal Pod Autoscaler](evidencias/09-nivel-7-hpa.png)

---

## Estado final do cluster

Antes do teste de carga do HPA, registrei o estado dos recursos com:

```bash
kubectl get all -n desafio-kubernetes
```

Nesse momento o ambiente estava com:

- PostgreSQL com 1 Pod;
- PostgREST com 2 réplicas;
- Services `postgres` e `postgrest`;
- Deployments e ReplicaSets funcionando.

![Estado final do Kubernetes](evidencias/08-estado-final-kubernetes.png)

No teste do nível 7, o HPA aumentou temporariamente o PostgREST para 5 réplicas por causa da carga gerada.

---

## Como aplicar os manifests

Os principais arquivos podem ser aplicados nesta ordem:

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

> O Secret real precisa ser criado antes dos Deployments que dependem dele. O arquivo `03-secret.example.yaml` é apenas um modelo e não possui as credenciais utilizadas no ambiente.

## Comandos úteis

Alguns comandos que usei bastante durante o desafio:

```bash
kubectl get all -n desafio-kubernetes
kubectl get pods -n desafio-kubernetes
kubectl get services -n desafio-kubernetes
kubectl get pvc -n desafio-kubernetes
kubectl get hpa -n desafio-kubernetes
kubectl top pods -n desafio-kubernetes
```

Para investigar um Pod:

```bash
kubectl describe pod <nome-do-pod> -n desafio-kubernetes
```

Para consultar os logs:

```bash
kubectl logs <nome-do-pod> -n desafio-kubernetes
```

---

## Segurança

As credenciais reais não ficam no repositório.

O projeto disponibiliza somente:

```text
k8s/03-secret.example.yaml
```

com valores fictícios. Os arquivos locais usados para armazenar os Secrets reais são ignorados pelo `.gitignore`.

Antes dos commits, também é importante conferir o `git status` e garantir que nenhuma credencial entrou no versionamento por engano.

---

## O que pratiquei neste desafio

Durante o projeto trabalhei principalmente com:

- Namespace, Pods, Deployments e ReplicaSets;
- Services e DNS interno do Kubernetes;
- ConfigMaps e Secrets;
- PersistentVolumeClaims;
- persistência de dados;
- comunicação entre PostgREST e PostgreSQL;
- Liveness e Readiness Probes;
- requests e limits;
- escalabilidade horizontal;
- HPA e Metrics Server;
- métricas de CPU;
- geração de carga;
- troubleshooting;
- Git e GitHub.

## Conclusão

Esse desafio foi importante para sair um pouco da parte teórica e entender melhor o que acontece com uma aplicação rodando no Kubernetes.

Além de criar os manifests, consegui testar situações que ajudaram a visualizar melhor o funcionamento do cluster. Excluí Pods para observar o comportamento dos Deployments, validei a persistência dos dados do PostgreSQL, testei a comunicação da API com o banco e configurei as probes e os recursos do PostgREST.

A parte do HPA também foi interessante porque não funcionou tudo de primeira. Precisei resolver o problema do Metrics Server no ambiente local e, depois disso, consegui gerar carga e acompanhar o PostgREST aumentando de 2 para 5 réplicas automaticamente.

No final, o desafio me ajudou a entender melhor não só como criar os recursos do Kubernetes, mas também como verificar o que está acontecendo no cluster e investigar problemas quando alguma coisa não funciona como esperado.
