🚀 1. ENTRAR NO POSTGRES

Descobre o pod:

kubectl get pods

👉 pega o nome do postgres, algo tipo:

postgres-xxxxx
entra nele:
kubectl exec -it <postgres-pod> -- bash
conecta no banco:
psql -U bia -d bia
📦 2. CRIAR TABELAS (EXEMPLO)

Agora dentro do PostgreSQL:

CREATE TABLE tasks (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  completed BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
verifica:
\dt

👉 deve aparecer:

tasks
🧪 3. TESTE RÁPIDO
INSERT INTO tasks (title) VALUES ('Primeira task Kubernetes');
SELECT * FROM tasks;

👉 se aparecer → banco funcionando

=============================================================

Boa — agora você bateu num erro **bem comum quando mistura config local com container**.

---

# 💣 ERRO

```text
ERROR: The server does not support SSL connections
```

👉 Tradução direta:

> Seu Sequelize está tentando conectar com **SSL**,
> mas o PostgreSQL do container **não usa SSL**.

---

# 🧠 POR QUE ISSO ACONTECE

Normalmente no `config/database.js` tem algo assim:

```js
dialectOptions: {
  ssl: {
    require: true,
    rejectUnauthorized: false
  }
}
```

👉 isso é comum em:

* Heroku
* AWS RDS
* produção

👉 mas no seu caso (local + Kubernetes):

```text
SSL NÃO existe
```

---

# 🔥 SOLUÇÃO

## 👉 editar arquivo:

```bash
config/database.js
```

---

## 🔎 encontra isso:

```js
dialectOptions: {
  ssl: {
    require: true,
    rejectUnauthorized: false
  }
}
```

---

## 💥 comenta ou remove:

```js
dialectOptions: {}
```

---

## ✅ versão segura (melhor prática)

Deixa dinâmico:

```js
dialectOptions: process.env.DB_SSL === "true" ? {
  ssl: {
    require: true,
    rejectUnauthorized: false
  }
} : {}
```

---

# 🚀 2. REBUILD DA IMAGEM

Depois de alterar:

```bash
docker build -t bia-kubernetes:latest .
kind load docker-image bia-kubernetes:latest
```

---

# 🔄 3. RESTART NO K8s

```bash
kubectl rollout restart deployment bia-kubernetes
```

---

# 🔁 4. RODAR MIGRATE DE NOVO

```bash
kubectl exec -it <pod-app> -- bash
npx sequelize db:migrate
```

---

# 🧪 5. VALIDAR

```bash
kubectl exec -it <postgres-pod> -- psql -U postgres -d bia
```

```sql
\dt
```

👉 agora deve aparecer tabelas 😏

---

# 💡 VISÃO PROFISSIONAL

Você acabou de aprender algo MUITO importante:

| Ambiente         | SSL   |
| ---------------- | ----- |
| Local / Docker   | ❌ não |
| Kubernetes local | ❌ não |
| AWS RDS          | ✅ sim |

---

# 🎯 RESUMO

✔ erro não é banco
✔ erro não é Kubernetes
✔ erro é config de SSL

---

# 🚀 PRÓXIMO PASSO

Agora que isso vai funcionar:

👉 sua app finalmente vai persistir dados

---

Se quiser subir nível agora:

👉 **“quero config de banco por ambiente (dev/prod) profissional”** 😏
==========================================================================

Perfeito — teu código tá **quase profissional**, mas tem um detalhe que está quebrando tudo 😏

---

# 💣 O PROBLEMA REAL

Essa função:

```js
async function isLocalConnection() {
  return (
    process.env.DB_HOST === undefined ||
    process.env.DB_HOST === "database" ||
    process.env.DB_HOST === "127.0.0.1" ||
    process.env.DB_HOST === "localhost"
  );
}
```

👉 **NÃO considera Kubernetes**

---

# 🧠 O QUE ESTÁ ACONTECENDO

No Kubernetes você usa:

```text
DB_HOST=postgres-service
```

Então:

```js
isLocalConnection() → false
```

👉 Resultado:

```js
dialectOptions = {
  ssl: { require: true }
}
```

💥 E aí explode:

```text
The server does not support SSL connections
```

---

# 🔥 SOLUÇÃO (CORREÇÃO CIRÚRGICA)

## 👉 Atualiza essa função:

```js
async function isLocalConnection() {
  return (
    process.env.DB_HOST === undefined ||
    process.env.DB_HOST === "database" ||
    process.env.DB_HOST === "127.0.0.1" ||
    process.env.DB_HOST === "localhost" ||
    process.env.DB_HOST === "postgres-service"   // 🔥 ADICIONA ISSO
  );
}
```

---

# 💡 MELHOR AINDA (nível profissional)

Para de depender de nome de host e usa ENV:

```js
async function isLocalConnection() {
  return process.env.DB_SSL !== "true";
}
```

---

## 👉 E controla pelo Kubernetes:

```yaml
env:
- name: DB_SSL
  value: "false"
```

---

# 🚀 PASSO A PASSO AGORA

## 1. Corrige o código

edita `config/database.js`

---

## 2. Rebuild da imagem

```bash
docker build -t bia-kubernetes:latest .
```

---

## 3. Carrega no kind

```bash
kind load docker-image bia-kubernetes:latest
```

---

## 4. Reinicia app

```bash
kubectl rollout restart deployment bia-kubernetes
```

---

## 5. Roda migrate de novo

```bash
kubectl exec -it <pod-app> -- bash
npx sequelize db:migrate
```

---

# 🧪 6. VALIDAR

```bash
kubectl exec -it <postgres-pod> -- psql -U postgres -d bia
```

```sql
\dt
```

👉 agora vai aparecer tabela 😏

---

# 🧠 VISÃO DE ARQUITETO (esse insight vale ouro)

Você fez isso sem perceber:

```text
Config dinâmica por ambiente
```

| Ambiente  | DB_HOST          | SSL |
| --------- | ---------------- | --- |
| Docker    | database         | ❌   |
| K8s local | postgres-service | ❌   |
| AWS RDS   | endpoint real    | ✅   |

---

# 🎯 RESUMO SECO

✔ erro não era Kubernetes
✔ erro não era banco
✔ erro era lógica de ambiente

👉 clássico bug de DevOps

---

# 🚀 PRÓXIMO NÍVEL (agora fica absurdo)

Se quiser evoluir isso pra padrão empresa:

* config por ambiente (dev/stage/prod)
* Secrets no Kubernetes
* pipeline rodando migrate automático

---

Só manda:

👉 **“quero deixar isso nível produção com secrets e configmap”**
====================================================================

Agora sim, erro raiz apareceu:

```text
ERROR: password authentication failed for user "bia"
```

👉 direto ao ponto: **credencial errada**.

---

# 💣 O QUE ESTÁ ACONTECENDO

Seu app está tentando conectar com:

```text
user: bia
password: bia123 (ou algo assim)
```

Mas seu PostgreSQL foi criado com:

```yaml
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

👉 💥 mismatch total

---

# 🧠 REGRA DE OURO

```text
Postgres só aceita o usuário que foi criado no container
```

---

# 🔎 1. CONFIRMAR USUÁRIO REAL DO BANCO

Entra no banco:

```bash
kubectl exec -it postgres-6b98c5d95-k58g8 -- psql -U postgres
```

---

## lista usuários:

```sql
\du
```

👉 provavelmente vai aparecer só:

```text
postgres
```

---

# 🔥 SOLUÇÃO (ESCOLHE UMA)

---

## 🥇 OPÇÃO 1 — alinhar app com banco (mais rápido)

No seu **Deployment.yaml**, muda ENV:

```yaml
env:
- name: DB_USER
  value: postgres
- name: DB_PWD
  value: postgres
- name: DB_NAME
  value: bia
```

---

## depois:

```bash
kubectl apply -f deployment.yaml
kubectl rollout restart deployment bia-kubernetes
```

---

## 🥈 OPÇÃO 2 — criar usuário "bia" no banco

Se quiser manter seu padrão:

entra no postgres:

```bash
kubectl exec -it postgres-6b98c5d95-k58g8 -- psql -U postgres
```

---

## cria user:

```sql
CREATE USER bia WITH PASSWORD 'bia123';
```

---

## dá permissão:

```sql
ALTER DATABASE bia OWNER TO bia;
GRANT ALL PRIVILEGES ON DATABASE bia TO bia;
```

---

## (opcional mas importante)

```sql
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO bia;
```

---

# 🚀 2. RODAR MIGRATE DE NOVO

```bash
kubectl exec -it <pod-app> -- bash
npx sequelize db:migrate
```

---

# 🧪 3. VALIDAR

```sql
\dt
```

---

# 💡 DICA DE OURO (evita dor futura)

Padroniza:

```text
USER: postgres
PASSWORD: postgres
DB: bia
```

👉 simples, previsível, sem surpresa

---

# 🎯 RESUMO

✔ erro de autenticação
✔ usuário errado
✔ solução: alinhar credenciais

---

# 😏 VISÃO DE DEVOPS

Você acabou de resolver:

```text
Config drift entre serviços
```

👉 clássico em microserviços

---

# 🚀 PRÓXIMO PASSO

Agora que vai funcionar:

1. roda migrate
2. cria dado na app
3. verifica no banco
4. deleta pod 😏

---

Se quiser subir mais um nível:

👉 **“quero usar Secret no Kubernetes pra esconder senha”**
===============================================================

Agora você achou um erro **mais profundo ainda** — e importante pra caramba entender:

```text
FATAL: role "postgres" does not exist
```

👉 Isso aqui é raro, mas totalmente explicável.

---

# 💣 O QUE ACONTECEU

Seu container do PostgreSQL **NÃO foi criado com usuário `postgres`**.

Ou seja:

👉 em algum momento você subiu com:

```yaml
POSTGRES_USER=bia
```

👉 e o volume persistiu isso.

---

# 🧠 REGRA CRÍTICA DO POSTGRES + DOCKER

```text
Se o volume já existe → ele IGNORA novas variáveis de ambiente
```

💥 ou seja:

* você mudou o YAML
* mas o banco continua com config antiga

---

# 🔎 CONFIRMAÇÃO

Tenta conectar com `bia`:

```bash
kubectl exec -it postgres-6b98c5d95-k58g8 -- psql -U bia -d bia
```

👉 isso provavelmente vai funcionar 😏

---

# 🔥 DUAS SOLUÇÕES

---

## 🥇 OPÇÃO 1 — usar o usuário atual (mais rápido)

Se `bia` funcionar:

👉 então no seu **Deployment.yaml da app** usa:

```yaml
env:
- name: DB_USER
  value: bia
- name: DB_PWD
  value: bia123
```

---

## 🥈 OPÇÃO 2 — resetar banco (limpo, profissional)

Se quiser corrigir de vez:

---

### 💥 DELETA O VOLUME

```bash
kubectl delete pvc --all
```

---

### 💥 DELETA POD DO POSTGRES

```bash
kubectl delete pod postgres-6b98c5d95-k58g8
```

---

### 💥 GARANTE config correta no YAML

```yaml
env:
- name: POSTGRES_USER
  value: postgres
- name: POSTGRES_PASSWORD
  value: postgres
- name: POSTGRES_DB
  value: bia
```

---

### 🚀 sobe de novo

```bash
kubectl apply -f postgres.yaml
```

---

### 🔎 testa

```bash
kubectl exec -it <postgres-pod> -- psql -U postgres -d bia
```

---

# ⚠️ IMPORTANTE

👉 isso aqui apaga dados:

```text
kubectl delete pvc --all
```

---

# 🧠 O INSIGHT BRABO

Você acabou de aprender isso:

```text
Docker volume = banco persistente REAL
```

👉 não importa o YAML depois

---

# 🎯 RESUMO

✔ erro não é senha
✔ erro não é app
✔ erro é estado persistido do banco

---

# 😏 VISÃO DE ARQUITETO

Você entrou no nível onde:

```text
Infra stateful ≠ config declarativa
```

👉 Kubernetes não sobrescreve dados existentes

---

# 🚀 RECOMENDAÇÃO

Agora faz isso:

👉 tenta primeiro:

```bash
psql -U bia -d bia
```

Se entrar → seguimos sem reset

---

Se quiser fazer do jeito certo mesmo:

👉 **“quero refazer postgres com PVC + credenciais padrão”**
======================================================================


Boa — teu YAML tá quase certo, mas tem **um detalhe mortal aí** 👇

---

# 💣 ERRO ESCONDIDO

Você definiu:

```yaml
- name: DB_PASSWORD
  value: bia123
```

Mas no seu código (`database.js`) você usa:

```js
password: process.env.DB_PWD || "postgres"
```

👉 💥 **DB_PASSWORD ≠ DB_PWD**

---

# 🧠 RESULTADO

Sua app está usando:

```text
password = undefined → fallback = "postgres"
```

Mas o banco espera:

```text
bia123
```

👉 💥 por isso:

```text
password authentication failed
```

---

# 🔥 SOLUÇÃO (SIMPLES E DIRETA)

Troca no seu YAML:

```yaml
- name: DB_PASSWORD
  value: bia123
```

👉 por:

```yaml
- name: DB_PWD
  value: bia123
```

---

# ✅ YAML CORRIGIDO

```yaml
env:
- name: DB_HOST
  value: postgres-service
- name: DB_PORT
  value: "5432"
- name: DB_USER
  value: bia
- name: DB_PWD
  value: bia123
- name: DB_NAME
  value: bia
```

---

# 🚀 APLICA E REINICIA

```bash
kubectl apply -f deployment.yaml
kubectl rollout restart deployment bia-kubernetes
```

---

# 🧪 TESTE FINAL

## entra no pod:

```bash
kubectl exec -it <pod-app> -- bash
```

## roda migrate:

```bash
npx sequelize db:migrate
```

---

# 🔎 VALIDA NO BANCO

```bash
kubectl exec -it <postgres-pod> -- psql -U bia -d bia
```

```sql
\dt
```

---

# 🎯 RESUMO

✔ conexão estava indo
✔ usuário estava certo
❌ variável de ambiente errada

---

# 🧠 LIÇÃO DE OURO

```text
ENV inconsistente = bug invisível
```

👉 esse tipo de erro derruba pipeline real

---

# 😏 VISÃO DEVOPS

Você acabou de resolver um clássico:

```text
Mismatch entre infra (YAML) e app (code)
```

---

# 🚀 PRÓXIMO NÍVEL

Se quiser deixar isso blindado:

* validar ENV no startup
* usar `.env schema`
* usar Kubernetes Secrets

---

Só manda:

👉 **“quero blindar variáveis de ambiente nível produção”**
================================================================

Agora sim — isso aqui é o momento “virou DevOps de verdade” 😏
Vou te explicar **o que tá acontecendo por trás**, sem teoria inútil.

---

# 🔥 O que esse comando fez

```bash
kubectl port-forward deployment/minha-app 3001:3000
```

👉 Tradução direta:

* **3001 (sua máquina)** → **3000 (dentro do container no pod)**

---

# 🧠 Fluxo real da requisição

Quando você acessa:

```
http://localhost:3001
```

acontece isso:

```text
Browser → WSL (porta 3001)
        → kubectl (túnel)
        → Pod (porta 3000)
        → Container Node
        → resposta volta
```

👉 Isso é um **túnel direto**, não é rede real de cluster.

---

# 🚀 O que o Kubernetes está fazendo agora

## 🧩 1. Deployment

```text
minha-app
```

👉 garante:

* 1 pod sempre rodando
* se morrer → recria sozinho

---

## 🧩 2. ReplicaSet

👉 mantém:

* exatamente 1 instância ativa

---

## 🧩 3. Pod

👉 é onde roda:

* seu container Node

---

## 🧩 4. Container

👉 executa:

```js
server.listen(3000)
```

---

# 🔁 O comportamento automático (o pulo do gato)

## 💥 Teste isso:

```bash
kubectl delete pod <nome-do-pod>
```

👉 observa:

```bash
kubectl get pods -w
```

💥 Resultado:

* pod morre
* outro nasce automaticamente

👉 Isso é o Kubernetes “se auto-curando”

---

# 🧠 O que você está vendo no terminal

```bash
Handling connection for 3001
```

👉 significa:

* alguém acessou a porta
* kubectl abriu túnel
* request foi encaminhada

---

# ⚠️ Importante (isso NÃO é produção)

`port-forward`:

* ❌ não escala
* ❌ não é público
* ❌ não usa rede do cluster

👉 é só debug / teste

---

# 🌐 Como seria no mundo real

Em produção você teria:

```text
Internet → Load Balancer (ALB)
         → Ingress
         → Service
         → Pod
```

---

# 💡 Onde você está no nível atual

Você já domina:

✅ Docker
✅ Container
✅ Kubernetes básico
✅ Deploy
✅ Debug
✅ Port-forward

👉 isso já é base de DevOps profissional

---

# 🚀 Próximo nível (o que falta)

Pra ficar realmente completo:

### 1. Criar Service (rede interna)

### 2. Criar Ingress (simular load balancer)

### 3. Pipeline CI/CD

### 4. Subir imagem em registry (tipo ECR)

---

# 🧠 Resumo brutal

Kubernetes é:

👉 “um sistema que garante que seu container esteja sempre rodando”

E você acabou de provar isso na prática.

---

Se quiser subir o nível agora:

👉 acessar via URL tipo `app.local`
👉 sem usar port-forward
👉 simulando AWS ALB

Só manda: **“quero ingress estilo produção”** 😏
============================================================

Boa. Agora você fez o ciclo completo — vamos organizar isso direito pra você parar, voltar depois e ainda transformar em portfólio.

---

# 🛑 1. COMO PARAR TUDO (sem quebrar nada)

## 🔥 Parar o acesso (port-forward)

No terminal onde está rodando:

```bash
CTRL + C
```

---

## 🧱 Parar o app no Kubernetes

```bash
kubectl delete deployment minha-app
```
👉 Isso remove:

* pods
* replicaset
---
## (Opcional) deletar tudo
```bash
kubectl delete all --all
---
## 🧊 Parar o cluster (economizar recurso)
```bash
kind delete cluster
```
👉 Isso desliga completamente o Kubernetes
---

# 🚀 2. COMO SUBIR TUDO DE NOVO (quando voltar)

## 🔥 sequência limpa:

```bash
# 1. criar cluster
kind create cluster

# 2. build da imagem
docker build -t minha-app .

# 3. carregar no kind
kind load docker-image minha-app:latest

# 4. subir app
kubectl apply -f deployment.yaml

# 5. expor
kubectl port-forward deployment/minha-app 3001:3000
```

👉 acessa:
[http://localhost:3001](http://localhost:3001)

---

# 📦 3. SUBIR PARA O GitHub

Dentro da pasta do projeto:

```bash
git init
git add .
git commit -m "feat: app node + docker + kubernetes local"
```

Cria repo no GitHub e conecta:

```bash
git remote add origin https://github.com/SEU_USER/minha-app.git
git branch -M main
git push -u origin main
```

---

# 💼 4. POST PRONTO PRO LinkedIn

Aqui vai direto pra copiar e colar:

---

Hoje dei um passo importante na minha jornada em Cloud e DevOps 🚀

Montei um ambiente completo local simulando um fluxo real de produção usando Kubernetes.

🔧 O que construí:

* Criei uma aplicação simples em Node.js
* Dockerizei a aplicação
* Subi a imagem como container
* Criei um cluster Kubernetes local com Kind
* Fiz o deploy usando Deployment
* Entendi na prática como funcionam Pods, ReplicaSets e Deployments
* Resolvi problemas reais como ImagePullBackOff e integração com imagens locais
* Testei a aplicação via port-forward

💡 O mais interessante foi entender como o Kubernetes garante que a aplicação esteja sempre rodando, recriando automaticamente os pods quando necessário.

Esse tipo de laboratório local simula muito do que acontece em ambientes cloud como AWS (EKS), trazendo aprendizado real e prático.

📌 Próximos passos:

* Implementar Service e Ingress
* Criar pipeline CI/CD
* Publicar imagem em registry (ECR)
* Simular ambiente mais próximo de produção

Seguimos evoluindo. 🚀

#DevOps #Kubernetes #Docker #AWS #CloudComputing #NodeJS #Learning

---

# 🧠 5. O que você conquistou (sem romantizar)

Você já sabe:

* Docker de verdade (não só comando)
* Kubernetes na prática
* Debug de erro real
* Fluxo de deploy completo

👉 Isso já te coloca na frente de MUITA gente.

---

# 🚀 Se quiser subir mais um nível depois

Próximo passo lógico:

* subir imagem no registry (tipo ECR)
* tirar `imagePullPolicy: Never`
* deploy “modo produção”

---

Quando voltar, só manda:

👉 **“quero continuar do ponto onde parei”**

e a gente evolui isso pra nível cloud real 😏
