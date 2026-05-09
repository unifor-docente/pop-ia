# POP-IA — Gerador de Procedimentos Operacionais Padrão com IA

Gera POPs no padrão ISO 9001 a partir de uma conversa em linguagem natural, usando Amazon Bedrock (Claude). Produz documento, diagrama de fluxo, BPMN 2.0 e PDF prontos para uso.

---

## Diagramas de Arquitetura

### C4 — Nível 1: Contexto

```mermaid
C4Context
  title Contexto do Sistema — POP-IA

  Person(usuario, "Usuário", "Descreve o processo a documentar via chat")

  System(pop_ia, "POP-IA", "Gera Procedimentos Operacionais Padrão estruturados com IA")

  System_Ext(bedrock, "Amazon Bedrock", "Modelo Claude 3.5 Haiku — geração de conteúdo estruturado")

  Rel(usuario, pop_ia, "Descreve o processo, anexa documentos")
  Rel(pop_ia, bedrock, "Invoca modelo de linguagem", "HTTPS / InvokeModel")
```

---

### C4 — Nível 2: Containers

```mermaid
C4Container
  title Containers — POP-IA

  Person(usuario, "Usuário")

  System_Boundary(aws, "AWS — us-east-1") {

    Container(cloudfront, "CloudFront", "CDN AWS", "Distribui o frontend via HTTPS com cache global")
    Container(s3_web, "S3 Web", "Object Storage", "Hospeda os arquivos estáticos do frontend")
    Container(apigw, "API Gateway", "HTTP API", "Expõe endpoint HTTPS para geração de POPs")
    Container(lambda, "Lambda", "Node.js 20 · ARM64 · 1 GB · 120 s", "Orquestra prompt, IA, normalização, renderização e persistência")
    ContainerDb(s3_docs, "S3 Documentos", "Object Storage", "Armazena JSON, Mermaid, BPMN, HTML e PDF gerados")
    ContainerDb(dynamo, "DynamoDB", "NoSQL · PAY_PER_REQUEST", "Histórico de gerações com checksum")

  }

  System_Ext(bedrock, "Amazon Bedrock", "Claude Haiku 4.5\nus.anthropic.claude-haiku-4-5-20251001-v1:0")

  Rel(usuario, cloudfront, "Acessa o frontend", "HTTPS")
  Rel(cloudfront, s3_web, "Serve arquivos estáticos", "OAC / S3 privado")
  Rel(usuario, apigw, "Envia contexto do processo", "POST / JSON")
  Rel(apigw, lambda, "Invoca função", "AWS Proxy v2")
  Rel(lambda, bedrock, "Gera conteúdo estruturado", "InvokeModel / HTTPS")
  Rel(lambda, s3_docs, "Salva artefatos", "PutObject / AWS SDK")
  Rel(lambda, dynamo, "Registra metadados", "PutItem / AWS SDK")
```

---

### C4 — Nível 3: Componentes da Lambda

```mermaid
C4Component
  title Componentes — Lambda Node.js

  Container_Boundary(lambda, "Lambda — apps/api/src") {

    Component(handler, "index.js", "Handler", "Roteamento, validação de entrada e orquestração do fluxo")
    Component(prompt, "prompt.js", "Prompt Builder", "Monta o prompt ISO 9001 com contexto da conversa e anexos")
    Component(bedrock_c, "bedrock.js", "Bedrock Client", "Invoca o modelo via AWS SDK, detecta inference profile")
    Component(schema, "schema.js", "Normalizer", "Valida e normaliza o JSON retornado pelo modelo")
    Component(pdf_r, "renderers/pdf.js", "PDF Renderer", "Gera PDF formatado com PDFKit")
    Component(bpmn_r, "renderers/bpmn.js", "BPMN Renderer", "Gera BPMN 2.0 XML")
    Component(mermaid_r, "renderers/mermaid.js", "Mermaid Renderer", "Gera diagrama de fluxo Mermaid")
    Component(html_r, "renderers/html.js", "HTML Renderer", "Gera relatório HTML standalone")
    Component(storage, "storage.js", "Storage", "Persiste artefatos no S3 e metadados no DynamoDB")

  }

  Rel(handler, prompt, "buildPrompt(payload)")
  Rel(handler, bedrock_c, "generateProcedureJson(prompt)")
  Rel(bedrock_c, schema, "parseModelJson(text)")
  Rel(handler, schema, "normalizeProcedure(output, payload)")
  Rel(handler, mermaid_r, "toMermaid(procedure)")
  Rel(handler, bpmn_r, "toBpmnXml(procedure)")
  Rel(handler, html_r, "toHtml(procedure, mermaid)")
  Rel(handler, pdf_r, "toPdfBuffer(procedure)")
  Rel(handler, storage, "persistArtifacts({...})")
```

---

### Arquitetura AWS

```mermaid
flowchart TB
  user(["👤 Usuário"])

  subgraph CDN ["CloudFront — CDN"]
    cf["🌐 Distribution\nHTTPS · redirect-to-https\nCache Policy: CachingOptimized"]
  end

  subgraph S3W ["S3 — Web (privado + OAC)"]
    s3w["📄 index.html\n📜 app.js\n🎨 styles.css\n⚙️ config.js"]
  end

  subgraph APIGW ["API Gateway — HTTP API"]
    apigw["🔀 POST /\nCORS habilitado\nAuto-deploy"]
  end

  subgraph LAMBDA ["Lambda — Node.js 20 · ARM64 · 120 s"]
    direction TB
    handler["⚡ index.js\nHandler"]
    subgraph CORE ["Núcleo"]
      prompt_m["📝 prompt.js"]
      bedrock_m["🤖 bedrock.js"]
      schema_m["🔧 schema.js"]
    end
    subgraph RENDERERS ["Renderizadores"]
      pdf_m["📄 pdf.js"]
      bpmn_m["📊 bpmn.js"]
      mmd_m["📈 mermaid.js"]
      html_m["🌐 html.js"]
    end
    storage_m["💾 storage.js"]
    handler --> CORE
    handler --> RENDERERS
    handler --> storage_m
  end

  subgraph BEDROCK ["Amazon Bedrock"]
    model["🧠 Claude Haiku 4.5\nus.anthropic.claude-haiku-4-5-20251001-v1:0\nInference Profile"]
  end

  subgraph S3D ["S3 — Documentos (privado)"]
    docs["📦 procedures/\n  processo.json\n  processo.mmd\n  processo.bpmn\n  relatorio.html\n  relatorio.pdf"]
  end

  subgraph DDB ["DynamoDB"]
    table["🗄️ procedures\nid · processName · code\nrevision · checksum\nPAY_PER_REQUEST"]
  end

  subgraph IAM ["IAM Role — Lambda"]
    iam["🔐 bedrock:InvokeModel\ns3:PutObject · s3:GetObject\ndynamodb:PutItem\nlogs:CreateLogGroup"]
  end

  user -->|"HTTPS"| cf
  cf -->|"OAC · S3 privado"| s3w
  user -->|"POST / JSON"| apigw
  apigw -->|"AWS Proxy v2"| handler
  bedrock_m -->|"InvokeModel"| model
  storage_m -->|"PutObject"| docs
  storage_m -->|"PutItem"| table
  LAMBDA -.->|"assume role"| IAM
```

---

## Fluxo de Geração

```mermaid
sequenceDiagram
  actor U as Usuário
  participant FE as Frontend
  participant GW as API Gateway
  participant LB as Lambda
  participant BR as Bedrock
  participant S3 as S3 Docs
  participant DB as DynamoDB

  U->>FE: Descreve o processo via chat
  U->>FE: Clica em "Gerar POP"
  FE->>GW: POST / {processName, messages, attachments}
  GW->>LB: Invoca handler (AWS Proxy v2)
  LB->>LB: Valida e normaliza entrada
  LB->>LB: buildPrompt() — monta prompt ISO 9001
  LB->>BR: InvokeModel (Claude Haiku 4.5)
  BR-->>LB: JSON estruturado do POP
  LB->>LB: normalizeProcedure() — garante schema
  LB->>LB: Gera Mermaid · BPMN · HTML · PDF
  LB->>S3: PutObject (5 artefatos)
  LB->>DB: PutItem (metadados)
  LB-->>GW: {procedure, artifacts, assets}
  GW-->>FE: 200 OK
  FE-->>U: Exibe documento, diagrama e downloads
```

---

## Stack

| Camada | Tecnologia |
|---|---|
| Frontend | HTML · CSS · JavaScript (vanilla) · Mermaid.js |
| CDN | Amazon CloudFront + S3 (OAC) |
| API | Amazon API Gateway HTTP API |
| Backend | AWS Lambda · Node.js 20 · ARM64 · 120 s |
| IA | Amazon Bedrock — Claude Haiku 4.5 (inference profile) |
| Documentos | Amazon S3 |
| Metadados | Amazon DynamoDB (PAY_PER_REQUEST) |
| IaC | Terraform >= 1.5 · AWS Provider ~> 5.0 |
| CI/CD | GitHub Actions |

---

## Outputs gerados por POP

| Arquivo | Formato | Descrição |
|---|---|---|
| `processo.json` | JSON | Schema estruturado do POP |
| `processo.mmd` | Mermaid | Diagrama de fluxo |
| `processo.bpmn` | BPMN 2.0 XML | Processo para editores BPMN |
| `relatorio.html` | HTML | Relatório standalone |
| `relatorio.pdf` | PDF | Documento final formatado |

---

## Rodar localmente (modo demo)

```bash
# Instalar dependências
npm install --prefix apps/api

# Terminal 1 — API (sem Bedrock, usa fallback)
npm run api:local

# Terminal 2 — Frontend
npm run web:local
```

Acesse `http://localhost:5173`.

---

## Deploy na AWS

### Pré-requisitos (uma única vez)

```bash
# 1. Configurar credenciais AWS
aws configure

# 2. Criar bucket de estado do Terraform
./scripts/bootstrap.sh

# 3. Preencher variáveis
cp infra/terraform.tfvars.example infra/terraform.tfvars
# editar: definir bedrock_model_id

# 4. Habilitar o modelo no console AWS
# Bedrock → Model catalog → Claude Haiku 4.5 → Subscribe/Enable
```

### Comandos

```bash
make deploy    # sobe toda a infraestrutura (~15 min pela CloudFront)
make update    # atualiza apenas o código da Lambda e frontend
make destroy   # remove tudo da AWS após a demonstração
make local     # roda API + frontend localmente
```

### GitHub Actions (CI/CD)

| Workflow | Trigger | Descrição |
|---|---|---|
| **Deploy** | Manual (Run workflow) | Build + terraform apply |
| **Destroy** | Manual + confirmação `DESTROY` | terraform destroy |

**Secrets necessários no repositório:**

| Secret | Valor |
|---|---|
| `AWS_ACCESS_KEY_ID` | Chave de acesso AWS |
| `AWS_SECRET_ACCESS_KEY` | Chave secreta AWS |
| `AWS_REGION` | `us-east-1` |
| `TF_STATE_BUCKET` | Nome do bucket gerado pelo `bootstrap.sh` |
| `BEDROCK_MODEL_ID` | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |

---

## Como obter bons resultados

A **estrutura** do POP é sempre a mesma (template ISO 9001 fixo em `prompt.js`). O que varia é o **conteúdo**, que depende do que o usuário descreve no chat.

| Entrada | Resultado |
|---|---|
| Campo "Processo" vazio + chat vazio | POP genérico com etapas-padrão ("Processo não identificado") |
| Nome do processo preenchido | Claude infere etapas prováveis para aquele processo |
| Descrição detalhada no chat | POP fiel ao processo real descrito |
| Anexo com fluxo, formulário ou política | POP enriquecido com o conteúdo do documento |

**Dica para a demonstração:** antes de clicar em "Gerar POP", peça ao aluno para descrever no chat as etapas principais, quem é o responsável, quais documentos são gerados e quais são as exceções do processo. Quanto mais contexto, melhor o resultado.

---

## Configurações de geração (Bedrock)

| Variável de ambiente | Valor padrão | Descrição |
|---|---|---|
| `BEDROCK_MODEL_ID` | — | Inference profile ID do modelo no Bedrock |
| `BEDROCK_MAX_TOKENS` | `3000` | Limite de tokens gerados por chamada |
| `BEDROCK_TEMPERATURE` | `0.2` | Grau de criatividade (0 = determinístico) |

> **Por que 3000 tokens?** O API Gateway HTTP API tem timeout fixo de **30 segundos**. Com 6000 tokens (padrão original) o modelo levava 30–36 s e retornava `503 Service Unavailable`. Com 3000 tokens a resposta chega em ~15 s. Para processos simples 3000 tokens é suficiente; para processos muito complexos é possível aumentar com um deploy local (`make deploy`) ou ajustando o secret `BEDROCK_MAX_TOKENS`.

---

## Divisão da equipe

| Papel | Responsável | Arquivos |
|---|---|---|
| Frontend | Aluno 1 | `apps/web/` |
| Backend IA | Aluno 2 | `apps/api/src/bedrock.js` · `prompt.js` · `schema.js` |
| Renderizadores | Aluno 3 | `apps/api/src/renderers/` |
| Infraestrutura | Integração | `infra/` · `scripts/` · `.github/` |

**Entrega: 06/05/2026**
Alteraro por Rafael