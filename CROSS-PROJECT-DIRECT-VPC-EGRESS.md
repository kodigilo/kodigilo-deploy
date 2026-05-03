# Direct VPC Egress cross-project via Shared VPC

Como configurar uma Cloud Function (Gen2) ou Cloud Run service em um projeto GCP (ex: QA, SANDBOX) para acessar um Cloud SQL com **IP privado** que vive em **outro projeto** (ex: DEV), usando **Direct VPC Egress** (sem VPC Connector).

## Contexto e problema

**Cenário típico:** monorepo de microserviços Firebase Functions com 4 ambientes (dev, qa, sandbox, prod). O banco de DEV é compartilhado entre dev/qa/sandbox (custos baixos), enquanto prod tem o seu próprio. Cada ambiente é um projeto GCP separado.

**O que NÃO funciona:**

VPC Peering simples não é transitivo. Se você fizer:

```
qa-vpc ↔ default DEV ↔ servicenetworking (banco)
```

A function em QA com Direct VPC Egress consegue **mandar** o pacote para `10.x.x.x` do banco, mas o pacote de **resposta não consegue voltar**, porque o servicenetworking peering só conhece subnets diretas da VPC peered (default DEV) — não subnets transitivas (qa-vpc).

Isso explica por que **VPC Connector "funciona" e Direct VPC Egress "não funciona"** nesse cenário: o connector tem mecanismo de proxy interno do Google que faz o caminho de retorno funcionar; Direct VPC Egress é roteamento puro, e portanto sofre da limitação de não-transitividade.

**O que funciona:** Shared VPC. A function de QA passa a usar uma subnet que vive **na própria VPC do banco** (default DEV). Não há mais peering envolvido — é a mesma VPC. Cross-project funciona porque o Google tem suporte oficial via Shared VPC.

## Pré-requisitos

- A organização GCP já tem Shared VPC configurado: o **host project** é o mesmo onde mora o Cloud SQL (ex: DEV), e os **service projects** são os ambientes consumidores (ex: QA, SANDBOX).
  ```bash
  # Verificar host
  gcloud compute shared-vpc get-host-project <PROJECT_QA>
  # Deve retornar o nome do host project (ex: gestio-school-dev)
  ```
- O workflow de deploy usa `gcloud functions deploy --gen2` (sem `--vpc-connector`) seguido de `gcloud run services update --network=... --subnet=... --vpc-egress=private-ranges-only --clear-vpc-connector`.
- Permissão de `Compute Network Admin` no host project para criar subnets e conceder IAM nelas.

## Passos

Substituir os placeholders pelos valores do seu projeto:

| Placeholder | Significado |
|---|---|
| `<HOST>` | Project ID do host (onde o banco mora). Ex: `gestio-school-dev` |
| `<QA>` | Project ID do consumidor 1. Ex: `gestio-school-qa` |
| `<SANDBOX>` | Project ID do consumidor 2. Ex: `gestio-school-sandbox` |
| `<NUM_QA>` | Project number numérico de `<QA>` (`gcloud projects describe <QA> --format='value(projectNumber)'`) |
| `<NUM_SANDBOX>` | Project number numérico de `<SANDBOX>` |
| `<REGION>` | Região das functions. Ex: `us-central1` |

### 1. Criar subnets dedicadas no host

Na VPC `default` do host project, criar uma subnet `/26` (mínimo aceito por Direct VPC Egress) por projeto consumidor.

**Atenção sobre CIDRs:** se a `default` está em "auto subnet mode" (padrão), o range `10.128.0.0/9` está reservado pra subnets auto-criadas em outras regiões. Use CIDRs FORA desse bloco — por exemplo `10.10.0.0/26`, `10.11.0.0/26`, ou ranges em `172.16.0.0/12` que não conflitem com nada.

```bash
gcloud compute networks subnets create qa-functions-<REGION> \
  --network=default --region=<REGION> --range=10.10.0.0/26 \
  --project=<HOST>

gcloud compute networks subnets create sandbox-functions-<REGION> \
  --network=default --region=<REGION> --range=10.11.0.0/26 \
  --project=<HOST>
```

### 2. Conceder IAM cross-project (`compute.networkUser`)

Pra cada subnet, dois service accounts do projeto consumidor precisam de `roles/compute.networkUser`:

- **Runtime SA do Cloud Run:** `<NUM>-compute@developer.gserviceaccount.com` (default compute SA do projeto consumidor)
- **Serverless Robot:** `service-<NUM>@serverless-robot-prod.iam.gserviceaccount.com` (faz binding da subnet no deploy)

```bash
# QA → subnet qa-functions-<REGION>
gcloud compute networks subnets add-iam-policy-binding qa-functions-<REGION> \
  --region=<REGION> --project=<HOST> \
  --member="serviceAccount:<NUM_QA>-compute@developer.gserviceaccount.com" \
  --role="roles/compute.networkUser"

gcloud compute networks subnets add-iam-policy-binding qa-functions-<REGION> \
  --region=<REGION> --project=<HOST> \
  --member="serviceAccount:service-<NUM_QA>@serverless-robot-prod.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"

# SANDBOX → subnet sandbox-functions-<REGION>
gcloud compute networks subnets add-iam-policy-binding sandbox-functions-<REGION> \
  --region=<REGION> --project=<HOST> \
  --member="serviceAccount:<NUM_SANDBOX>-compute@developer.gserviceaccount.com" \
  --role="roles/compute.networkUser"

gcloud compute networks subnets add-iam-policy-binding sandbox-functions-<REGION> \
  --region=<REGION> --project=<HOST> \
  --member="serviceAccount:service-<NUM_SANDBOX>@serverless-robot-prod.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

### 3. Configurar o deploy para usar paths cross-project

No comando `gcloud run services update` (executado depois de `gcloud functions deploy --gen2`), passar os paths completos `projects/<HOST>/...`:

```bash
gcloud run services update <FUNCTION_NAME> \
  --region=<REGION> \
  --project=<QA> \
  --network=projects/<HOST>/global/networks/default \
  --subnet=projects/<HOST>/regions/<REGION>/subnetworks/qa-functions-<REGION> \
  --vpc-egress=private-ranges-only \
  --clear-vpc-connector
```

Mesma coisa para SANDBOX, trocando `<QA>` por `<SANDBOX>` e a subnet correspondente.

**Detalhe importante:** Cloud Run service inherits o nome da function, mas converte `_` em `-`. Se o `nomeDaFuncao` é `v1_atendimento`, o Cloud Run service se chama `v1-atendimento`. Use `tr '_' '-'` na pipeline para gerar o nome correto.

### 4. Centralizar valores em secrets (opcional mas recomendado)

Se o workflow lê os valores de network/subnet do Secret Manager (em vez de hardcoded), basta criar/atualizar 2 secrets por projeto consumidor com os paths cross-project:

```bash
# QA
echo -n "projects/<HOST>/global/networks/default" | \
  gcloud secrets versions add QA_NETWORK --data-file=- --project=<QA>
echo -n "projects/<HOST>/regions/<REGION>/subnetworks/qa-functions-<REGION>" | \
  gcloud secrets versions add QA_SUBNET --data-file=- --project=<QA>

# SANDBOX
echo -n "projects/<HOST>/global/networks/default" | \
  gcloud secrets versions add SANDBOX_NETWORK --data-file=- --project=<SANDBOX>
echo -n "projects/<HOST>/regions/<REGION>/subnetworks/sandbox-functions-<REGION>" | \
  gcloud secrets versions add SANDBOX_SUBNET --data-file=- --project=<SANDBOX>
```

Se os secrets ainda não existem, trocar `versions add` por `create --data-file=-`.

## Validação

Após deploy:

1. **Verificar que o Cloud Run está com a config correta:**
   ```bash
   gcloud run services describe <FUNCTION_NAME> --region=<REGION> --project=<QA> \
     --format="value(spec.template.metadata.annotations)" | tr ',' '\n' | grep -iE "network|subnet|egress|vpc"
   ```
   Deve mostrar `network-interfaces=[{...}]` apontando pro projeto host, e **não** `vpc-access-connector`.

2. **Chamar uma rota da function que faz query no banco** — deve retornar dados sem erro de "Can't reach database server".

3. **Logs:** se houver erro de conectividade, vai aparecer como `PrismaClientInitializationError: Can't reach database server at <IP>:3306` (ou similar pro driver que estiver usando).

## Custos

- **Subnets:** gratuitas
- **Direct VPC Egress:** gratuito (escala junto com as instâncias)
- **Comparado a VPC Connector:** economia de ~R$50/mês por projeto consumidor (cada connector usa 2 e2-micro 24/7)

## Reversão

Pra voltar pra VPC Connector:

```bash
gcloud run services update <FUNCTION_NAME> --region=<REGION> --project=<QA> \
  --vpc-connector=<NOME_DO_CONNECTOR_ANTIGO> \
  --clear-network --vpc-egress=private-ranges-only
```

E nos secrets, voltar `QA_NETWORK`/`QA_SUBNET` aos valores antigos (ou desabilitar a versão nova com `gcloud secrets versions disable <NEW_VERSION>`).

## Por que não usei outras opções

- **VPC Connector:** funciona mas custa ~R$50/mês por projeto e tem latência adicional do proxy.
- **Private Service Connect (PSC) for Cloud SQL:** funciona cross-project nativamente, mas custa ~R$36/mês por endpoint e adiciona um recurso a mais pra manter. Vale considerar se o Shared VPC não estiver disponível.
- **Bancos próprios em cada projeto:** elimina cross-project mas duplica esforço de schema/backup/dados.
- **Re-criar VPCs ou peerings com configurações diferentes:** não resolve. O problema é a transitividade do servicenetworking peering, que é por design do Google.

## Referências

- [Direct VPC egress with a Shared VPC network](https://cloud.google.com/run/docs/configuring/shared-vpc-direct-vpc)
- [Configure Direct VPC egress for 2nd gen functions](https://cloud.google.com/functions/docs/running/direct-vpc)
- [Compare Direct VPC egress and VPC connectors](https://cloud.google.com/run/docs/configuring/connecting-vpc)
