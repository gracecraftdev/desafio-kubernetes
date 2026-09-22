# Desafio Kubernetes — Fundamentos na Prática

Projeto desenvolvido durante o **CloudOps Bootcamp**, com o objetivo de praticar os principais fundamentos do Kubernetes por meio da implantação de uma API integrada a um banco de dados PostgreSQL.

A aplicação utiliza o **PostgREST** para disponibilizar uma API REST conectada ao PostgreSQL dentro de um cluster Kubernetes local.

Durante o desafio foram utilizados recursos como Pods, Deployments, Services, ConfigMaps, Secrets, PersistentVolumeClaims, health checks, gerenciamento de recursos e escalabilidade.

---

## Arquitetura

A arquitetura implementada foi:

```text
Usuário
   |
   | HTTP
   v
Service PostgREST
   |
   v
Deployment PostgREST
   |
   | SQL
   v
Service PostgreSQL
   |
   v
Deployment PostgreSQL
   |
   v
PersistentVolumeClaim (PVC)
```

A comunicação entre a API e o banco de dados é realizada utilizando o nome do Service `postgres`.

Dessa forma, a aplicação não depende diretamente do IP do Pod do PostgreSQL, que pode mudar sempre que um Pod é recriado.

---

## Tecnologias utilizadas

- Kubernetes
- Docker Desktop
- kubectl
- PostgreSQL 16
- PostgREST
- YAML
- Git
- GitHub
- WSL2 / Ubuntu

---

## Estrutura do projeto

```text
desafio-kubernetes/
│
├── evidencias/
│   ├── 01-nivel-1-cluster-namespace-pod.png
│   ├── 02-nivel-1-inspecao-pod.png
│   ├── 03-nivel-1-exclusao-pod.png
│   ├── 04-niveis-2-3-postgresql-configuracao.png
│   ├── 05-nivel-4-api-postgrest.png
│   ├── 06-nivel-5-persistencia-pvc.png
│   ├── 07-nivel-6-health-recursos-scaling.png
│   └── 08-estado-final-kubernetes.png
│
├── k8s/
│   ├── 01-namespace.yaml
│   ├── 02-pod-teste.yaml
│   ├── 03-postgres-pvc.yaml
│   ├── 03-secret.example.yaml
│   ├── 04-configmap.yaml
│   ├── 05-postgres-deployment.yaml
│   ├── 06-postgres-service.yaml
│   ├── 07-postgrest-deployment.yaml
│   └── 08-postgrest-service.yaml
│
├── .gitignore
└── README.md
```

---

# Execução do projeto

## 1. Criando o Namespace

Todos os recursos da aplicação são executados no namespace:

```text
desafio-kubernetes
```

Para criá-lo:

```bash
kubectl apply -f k8s/01-namespace.yaml
```

Verifique:

```bash
kubectl get namespaces
```

---

## 2. Pod de teste

Inicialmente foi criado um Pod simples utilizando Nginx para observar o funcionamento e o ciclo de vida de um Pod no Kubernetes.

```bash
kubectl apply -f k8s/02-pod-teste.yaml
```

Verifique:

```bash
kubectl get pods -n desafio-kubernetes
```

Para visualizar informações detalhadas:

```bash
kubectl describe pod pod-teste -n desafio-kubernetes
```

Para visualizar os logs:

```bash
kubectl logs pod-teste -n desafio-kubernetes
```

Depois o Pod foi excluído:

```bash
kubectl delete pod pod-teste -n desafio-kubernetes
```

Como esse Pod foi criado diretamente, sem estar sendo gerenciado por um Deployment ou outro controlador, ele **não foi recriado automaticamente**.

Esse comportamento foi posteriormente comparado ao PostgreSQL, que é gerenciado por um Deployment e, portanto, tem seu Pod recriado automaticamente quando ocorre uma exclusão.

---

## 3. PersistentVolumeClaim

O PostgreSQL utiliza armazenamento persistente para que seus dados não sejam perdidos quando o Pod for excluído ou recriado.

Crie o PVC:

```bash
kubectl apply -f k8s/03-postgres-pvc.yaml
```

Verifique:

```bash
kubectl get pvc -n desafio-kubernetes
```

O PVC utilizado solicita:

```text
1Gi
```

de armazenamento.

O status esperado é:

```text
Bound
```

### PVC x emptyDir

Um volume `emptyDir` existe apenas enquanto o Pod existe.

Quando o Pod é removido, os dados armazenados nesse volume também são removidos.

O PersistentVolumeClaim permite que os dados tenham um ciclo de vida independente do Pod, possibilitando que um novo Pod utilize novamente o armazenamento persistente.

---

## 4. Secret

As credenciais reais utilizadas pelo PostgreSQL **não são armazenadas no repositório Git**.

O projeto contém apenas um modelo:

```text
k8s/03-secret.example.yaml
```

Para utilizar o projeto, crie uma cópia local:

```bash
cp k8s/03-secret.example.yaml k8s/03-secret.yaml
```

O arquivo possui a seguinte estrutura:

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

Substitua os valores de exemplo por valores locais antes de aplicar o Secret.

### Senhas com caracteres especiais

Caso a senha possua caracteres reservados em URLs, é necessário utilizar uma versão codificada na variável `PGRST_DB_URI`.

É possível gerar a versão codificada com Python:

```bash
python3 -c "import urllib.parse; print(urllib.parse.quote('SUA_SENHA_FORTE', safe=''))"
```

Utilize o resultado somente no campo da URI:

```text
postgres://desafio_user:SUA_SENHA_URL_ENCODED@postgres:5432/desafio_db
```

Depois aplique:

```bash
kubectl apply -f k8s/03-secret.yaml
```

O arquivo real:

```text
k8s/03-secret.yaml
```

não deve ser enviado ao repositório.

### Secret e Base64

Base64 **não é criptografia**.

Secrets ajudam a separar informações sensíveis dos manifests da aplicação, mas ambientes reais ainda exigem controles adicionais de segurança, como RBAC adequado e mecanismos seguros de gerenciamento de segredos.

---

## 5. ConfigMap

As configurações não sensíveis da aplicação são armazenadas em um ConfigMap.

Aplique:

```bash
kubectl apply -f k8s/04-configmap.yaml
```

O ConfigMap contém configurações como:

```text
POSTGRES_DB
PGRST_DB_SCHEMAS
PGRST_DB_ANON_ROLE
```

Assim, configurações comuns ficam separadas das credenciais sensíveis.

---

## 6. PostgreSQL

A imagem utilizada para o banco de dados é:

```text
postgres:16
```

Aplique o Deployment:

```bash
kubectl apply -f k8s/05-postgres-deployment.yaml
```

Depois aplique o Service:

```bash
kubectl apply -f k8s/06-postgres-service.yaml
```

Verifique os recursos:

```bash
kubectl get pods -n desafio-kubernetes
kubectl get svc -n desafio-kubernetes
kubectl get pvc -n desafio-kubernetes
```

O PostgreSQL é acessado internamente por meio do Service:

```text
postgres
```

na porta:

```text
5432
```

Dessa forma, outras aplicações do cluster utilizam o DNS interno do Kubernetes em vez de depender diretamente do IP do Pod.

---

## 7. Configuração do banco de dados

Para acessar o PostgreSQL:

```bash
kubectl exec -it deployment/postgres -n desafio-kubernetes -- \
  psql -U desafio_user -d desafio_db
```

Dentro do PostgreSQL, foi criado o papel utilizado pelo PostgREST:

```sql
CREATE ROLE web_anon NOLOGIN;
```

O usuário da aplicação recebeu o papel:

```sql
GRANT web_anon TO desafio_user;
```

Depois foi criada a tabela utilizada nos testes:

```sql
CREATE TABLE public.items (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
```

Foram concedidas as permissões necessárias:

```sql
GRANT USAGE ON SCHEMA public TO web_anon;

GRANT SELECT, INSERT, UPDATE, DELETE
ON public.items
TO web_anon;

GRANT USAGE, SELECT
ON SEQUENCE public.items_id_seq
TO web_anon;
```

Para visualizar as tabelas:

```sql
\dt
```

Para sair:

```text
\q
```

---

## 8. PostgREST

A API utiliza a imagem:

```text
postgrest/postgrest
```

Aplique o Deployment:

```bash
kubectl apply -f k8s/07-postgrest-deployment.yaml
```

Depois aplique o Service:

```bash
kubectl apply -f k8s/08-postgrest-service.yaml
```

Verifique:

```bash
kubectl get pods -n desafio-kubernetes -l app=postgrest
```

A conexão com o PostgreSQL utiliza:

```text
postgres
```

como hostname.

Isso corresponde ao nome do Service do banco de dados e evita a utilização direta do IP do Pod.

---

## 9. Acessando a API

Para acessar o PostgREST a partir da máquina local, utilize:

```bash
kubectl port-forward -n desafio-kubernetes svc/postgrest 3000:3000
```

Com o `port-forward` em execução, abra outro terminal.

### Consultando os registros

```bash
curl http://127.0.0.1:3000/items
```

Inicialmente, o retorno pode ser:

```json
[]
```

---

## 10. Inserindo dados pela API

Para inserir um registro:

```bash
curl -i -X POST http://127.0.0.1:3000/items \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"name":"dado-persistente"}'
```

A API deve retornar o registro criado.

Exemplo:

```json
[
  {
    "id": 1,
    "name": "dado-persistente"
  }
]
```

Consulte novamente:

```bash
curl http://127.0.0.1:3000/items
```

Resultado esperado:

```json
[
  {
    "id": 1,
    "name": "dado-persistente"
  }
]
```

Isso confirma a integração:

```text
Cliente
   ↓
PostgREST
   ↓
Service postgres
   ↓
PostgreSQL
```

---

# Teste de persistência

Um dos principais testes realizados foi verificar se os dados continuariam disponíveis após a exclusão do Pod do PostgreSQL.

Primeiro consulte os dados:

```bash
curl http://127.0.0.1:3000/items
```

Exemplo:

```json
[
  {
    "id": 1,
    "name": "dado-persistente"
  }
]
```

Identifique o Pod atual:

```bash
kubectl get pods -n desafio-kubernetes -l app=postgres
```

Exclua o Pod:

```bash
kubectl delete pod <NOME_DO_POD> -n desafio-kubernetes
```

Consulte novamente:

```bash
kubectl get pods -n desafio-kubernetes -l app=postgres
```

Como o PostgreSQL é gerenciado por um Deployment, o Kubernetes cria automaticamente um novo Pod.

Quando o novo Pod estiver:

```text
1/1 Running
```

consulte novamente a API:

```bash
curl http://127.0.0.1:3000/items
```

O mesmo registro continua disponível.

Também é possível verificar o PVC:

```bash
kubectl get pvc -n desafio-kubernetes
```

O PVC permanece:

```text
Bound
```

Isso demonstra que o dado não estava armazenado apenas no filesystem efêmero do Pod.

O armazenamento persistente possui um ciclo de vida separado, permitindo que o novo Pod do PostgreSQL continue utilizando os dados existentes.

---

# Health Checks

O Deployment do PostgREST possui verificações de saúde.

## Readiness Probe

A Readiness Probe verifica se a aplicação está pronta para receber requisições.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

Quando a readiness falha, o Pod pode continuar em execução, mas deixa de ser considerado pronto para receber tráfego pelo Service.

## Liveness Probe

A Liveness Probe verifica se o container continua ativo.

```yaml
livenessProbe:
  tcpSocket:
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
```

Caso a liveness falhe repetidamente conforme a política configurada pelo Kubernetes, o container pode ser reiniciado.

### Diferença entre Liveness e Readiness

**Liveness**

Responde à pergunta:

> O container continua funcionando?

É utilizada para identificar situações em que o container precisa ser reiniciado.

**Readiness**

Responde à pergunta:

> O Pod está pronto para receber tráfego?

Ela controla se o Pod deve participar dos endpoints utilizados pelo Service.

---

# Requests e Limits

Foram configurados requests e limits para a API:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"

  limits:
    cpu: "200m"
    memory: "128Mi"
```

Os `requests` representam os recursos considerados pelo scheduler para a execução do Pod.

Os `limits` estabelecem os limites de utilização configurados para o container.

Essas configurações ajudam no gerenciamento dos recursos disponíveis no cluster.

---

# Escalabilidade

O Deployment do PostgREST foi configurado com:

```yaml
replicas: 2
```

Para verificar:

```bash
kubectl get deployment postgrest -n desafio-kubernetes
```

Exemplo:

```text
NAME        READY   UP-TO-DATE   AVAILABLE
postgrest   2/2     2            2
```

Também é possível visualizar os Pods:

```bash
kubectl get pods -n desafio-kubernetes -l app=postgrest
```

O Service identifica os Pods pelas labels e mantém os Pods prontos como backends.

Para visualizar os endpoints:

```bash
kubectl get endpointslices -n desafio-kubernetes \
  -l kubernetes.io/service-name=postgrest -o wide
```

Durante o teste, o Service possuía dois endpoints correspondentes às duas réplicas da API.

## Por que escalar a API e não simplesmente o PostgreSQL?

O PostgREST é utilizado de forma stateless nesse projeto.

Isso permite criar várias réplicas da API para atender requisições.

Já o PostgreSQL mantém estado persistente e utiliza um PVC `ReadWriteOnce`.

Por isso, simplesmente aumentar o número de réplicas do Deployment do PostgreSQL compartilhando o mesmo PVC não é equivalente a escalar uma aplicação stateless.

Bancos de dados exigem estratégias específicas de replicação e gerenciamento de estado.

---

# Versionamento com Git

O projeto foi versionado com Git durante o desenvolvimento.

As alterações foram separadas em commits de acordo com as principais etapas do desafio.

## Verificando alterações

```bash
git status
```

## Adicionando arquivos

Para adicionar arquivos específicos:

```bash
git add <arquivo>
```

Também é possível adicionar todas as alterações:

```bash
git add .
```

## Criando um commit

```bash
git commit -m "mensagem do commit"
```

## Enviando para o GitHub

```bash
git push
```

## Exemplos de commits realizados

Durante o desenvolvimento foram utilizados commits como:

```text
feat: add Kubernetes fundamentals and PostgreSQL setup
feat: add PostgREST API integration
test: validate PostgreSQL data persistence
feat: add health probes resources and API scaling
```

A separação dos commits por etapa facilita acompanhar a evolução do projeto e identificar quais recursos foram implementados em cada momento.

---

# Evidências

As evidências de execução estão disponíveis no diretório:

```text
evidencias/
```

## Nível 1 — Namespace e Pod

Criação do cluster, namespace e Pod:

![Namespace e Pod](evidencias/01-nivel-1-cluster-namespace-pod.png)

Inspeção do Pod:

![Inspeção do Pod](evidencias/02-nivel-1-inspecao-pod.png)

Exclusão do Pod:

![Exclusão do Pod](evidencias/03-nivel-1-exclusao-pod.png)

---

## Níveis 2 e 3 — PostgreSQL, PVC, ConfigMap e Secret

Configuração do PostgreSQL e dos recursos necessários:

![PostgreSQL, PVC, ConfigMap e Secret](evidencias/04-niveis-2-3-postgresql-configuracao.png)

---

## Nível 4 — Integração PostgREST + PostgreSQL

Teste da API realizando operações com o PostgreSQL:

![API PostgREST](evidencias/05-nivel-4-api-postgrest.png)

---

## Nível 5 — Persistência de dados

O mesmo registro é consultado antes e depois da exclusão do Pod do PostgreSQL.

O Deployment cria um novo Pod e o dado permanece disponível devido ao PVC.

![Persistência com PVC](evidencias/06-nivel-5-persistencia-pvc.png)

---

## Nível 6 — Probes, recursos e escalabilidade

Foram configurados:

- 2 réplicas do PostgREST;
- Readiness Probe;
- Liveness Probe;
- CPU requests;
- CPU limits;
- Memory requests;
- Memory limits;
- múltiplos endpoints no Service.

![Health Checks, recursos e scaling](evidencias/07-nivel-6-health-recursos-scaling.png)

---

## Estado final do cluster

Ao final da implementação, foi utilizado o comando:

```bash
kubectl get all -n desafio-kubernetes
```

para verificar o estado geral dos recursos executados no namespace.

O resultado confirma:

- PostgreSQL com `1/1` Pod em execução;
- PostgREST com `2/2` réplicas disponíveis;
- Services `postgres` e `postgrest`;
- Deployments disponíveis;
- ReplicaSets responsáveis pelo gerenciamento dos Pods.

![Estado final do Kubernetes](evidencias/08-estado-final-kubernetes.png)

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

Além da criação dos recursos, foram realizados testes de integração, exclusão e recriação de Pods, persistência dos dados e comunicação entre os componentes do cluster.

Ao final, o estado dos recursos foi verificado com `kubectl get all`, confirmando que os Pods, Services, Deployments e ReplicaSets necessários estavam em execução.