# Documentação do Processo ETL — `{NOME_DO_PROCESSO}`

> Template de documentação para pipelines ETL em **PySpark** orquestrados por **AWS Step Functions**, com infraestrutura **Terraform** e arquitetura **Data Mesh** (origens Hive/Iceberg → destino Iceberg).

---

## 1. Identificação

| Campo | Valor |
|-------|-------|
| **Nome do Processo** | `{nome_snake_case}` |
| **Domínio (Data Mesh)** | `{ex: vendas, financeiro, logistica}` |
| **Data Product** | `{nome_do_produto_de_dados}` |
| **Owner / Squad** | `{squad_responsavel}` |
| **Tech Lead** | `{nome}` |
| **Stakeholders de Negócio** | `{lista}` |
| **Criticidade** | `Baixa` / `Média` / `Alta` / `Crítica` |
| **SLA** | `{ex: D-1 até 07:00 BRT}` |
| **Frequência** | `{diária / horária / sob demanda}` |
| **Versão do Documento** | `1.0.0` |
| **Última Atualização** | `YYYY-MM-DD` |

---

## 2. Visão de Negócio

### 2.1 Objetivo

`{Descreva em 2-3 frases o problema de negócio que este ETL resolve. O que ele habilita? Qual decisão ou processo ele alimenta?}`

### 2.2 Consumidores do Data Product

| Consumidor | Uso | Frequência de Acesso |
|------------|-----|----------------------|
| `{ex: BI Vendas}` | `{ex: dashboard diário de receita}` | `{diária}` |
| `{ex: modelo ML churn}` | `{feature store}` | `{horária}` |

### 2.3 KPIs Impactados

- `{KPI 1}` — como é afetado
- `{KPI 2}` — como é afetado

---

## 3. Arquitetura Geral (Data Mesh)

### 3.1 Diagrama

```text
┌─────────────────── DOMÍNIO PRODUTOR ────────────────────┐
│                                                         │
│  Hive Table A ──┐                                       │
│  Hive Table B ──┼──▶ Step Function ──▶ Job PySpark ──┐  │
│  Iceberg Tbl C ─┘    (Orquestração)   (EMR/Glue)     │  │
│                                                       │  │
└──────────────────────────────────────────────────────┼──┘
                                                       │
                                                       ▼
                                          ┌──────────────────────┐
                                          │  Iceberg Destino     │
                                          │  {dominio}.{produto} │  ◀─── Data Contract
                                          └──────────────────────┘
                                                       │
                                                       ▼
                                          ┌──────────────────────┐
                                          │  Domínios Consumidores │
                                          └──────────────────────┘
```

### 3.2 Princípios Data Mesh Aplicados

| Princípio | Como é atendido |
|-----------|-----------------|
| **Domain Ownership** | `{qual domínio é dono deste produto?}` |
| **Data as a Product** | `{SLA, discoverability, qualidade garantida}` |
| **Self-Serve Platform** | `{como o pipeline usa a plataforma de dados comum}` |
| **Federated Governance** | `{políticas de catalogação, PII, classificação}` |

### 3.3 Camada na Arquitetura

- [ ] Raw / Bronze
- [ ] Trusted / Silver
- [ ] Refined / Gold
- [ ] Consumer-aligned (Data Product)

---

## 4. Fontes de Dados (Inputs)

### 4.1 Inventário de Tabelas Origem

| # | Tabela | Tipo | Database / Catalog | Domínio | Particionamento | Formato | Volume Aprox. | Owner |
|---|--------|------|--------------------|---------|-----------------|---------|---------------|-------|
| 1 | `{db.tabela_a}` | Hive | `{catalog}` | `{dominio_x}` | `dt=YYYY-MM-DD` | Parquet | `{linhas/dia}` | `{squad}` |
| 2 | `{db.tabela_b}` | Hive | `{catalog}` | `{dominio_x}` | `ano/mes/dia` | ORC | `{linhas/dia}` | `{squad}` |
| 3 | `{db.tabela_c}` | Iceberg | `{glue_catalog}` | `{dominio_y}` | `hidden: days(event_ts)` | Iceberg v2 | `{linhas/dia}` | `{squad}` |
| 4 | `{db.tabela_d}` | Iceberg | `{glue_catalog}` | `{dominio_z}` | `bucket(16, user_id)` | Iceberg v2 | `{linhas/dia}` | `{squad}` |

### 4.2 Contratos de Dados (Data Contracts)

Para cada fonte crítica, documente:

```yaml
source: {db.tabela_a}
owner: {squad_produtor}
schema:
  - name: id
    type: bigint
    nullable: false
  - name: event_ts
    type: timestamp
    nullable: false
  - name: {campo}
    type: {tipo}
sla:
  freshness: {ex: dados de D-1 disponíveis até 04:00 BRT}
  availability: 99.5%
pii_fields: [{lista_de_campos}]
breaking_changes_policy: {notificação com 30 dias}
```

### 4.3 Estratégia de Leitura

| Tabela | Estratégia | Filtro de Partição | Observação |
|--------|-----------|--------------------|-----------|
| `{db.tabela_a}` | Incremental por `dt` | `dt = '{reference_date}'` | Usa pushdown predicate |
| `{db.tabela_c}` | Incremental via Iceberg snapshot | `event_ts >= {watermark}` | Leitura por `start-snapshot-id` |
| `{db.tabela_d}` | Full scan | N/A | Tabela pequena (dimensão) |

---

## 5. Tabela Destino (Output)

### 5.1 Especificação da Tabela Iceberg

| Atributo | Valor |
|----------|-------|
| **Catalog** | `{ex: glue_catalog}` |
| **Database** | `{dominio}_{camada}` |
| **Tabela** | `{nome_tabela}` |
| **Formato** | Iceberg v2 |
| **Location** | `s3://{bucket}/iceberg/{dominio}/{tabela}/` |
| **Write Mode** | `append` / `overwrite` / `merge` (upsert) |
| **Particionamento** | `{ex: days(event_dt), bucket(8, customer_id)}` |
| **Sort Order** | `{ex: event_ts ASC}` |
| **Table Properties** | ver subseção 5.3 |

### 5.2 Schema da Tabela Destino

| Coluna | Tipo | Nullable | Descrição | PII |
|--------|------|----------|-----------|-----|
| `id` | `bigint` | N | Chave primária | N |
| `event_dt` | `date` | N | Data do evento (coluna de partição) | N |
| `customer_id` | `bigint` | N | ID do cliente | N |
| `{campo}` | `{tipo}` | `{S/N}` | `{descrição}` | `{S/N}` |

### 5.3 Table Properties (Iceberg)

```properties
format-version = 2
write.format.default = parquet
write.parquet.compression-codec = zstd
write.target-file-size-bytes = 536870912   # 512 MB
write.distribution-mode = hash
write.merge.mode = copy-on-write           # ou merge-on-read
commit.retry.num-retries = 4
history.expire.max-snapshot-age-ms = 604800000  # 7 dias
```

### 5.4 Estratégia de Manutenção

| Operação | Frequência | Responsável |
|----------|-----------|-------------|
| `expire_snapshots` | Diária | Job de manutenção `{nome}` |
| `rewrite_data_files` (compaction) | Semanal | Job de manutenção `{nome}` |
| `rewrite_manifests` | Semanal | Job de manutenção `{nome}` |
| `remove_orphan_files` | Mensal | Job de manutenção `{nome}` |

---

## 6. Lógica de Transformação

### 6.1 Fluxo de Alto Nível

```text
[READ src_a] ──┐
               ├──▶ [JOIN 1] ──▶ [FILTER] ──▶ [AGG 1] ──┐
[READ src_b] ──┘                                         │
                                                         ├──▶ [JOIN 3] ──▶ [WRITE Iceberg]
[READ src_c] ──┐                                         │
               ├──▶ [JOIN 2] ──▶ [WINDOW] ──▶ [AGG 2] ──┘
[READ src_d] ──┘
```

### 6.2 Etapas Detalhadas

Para **cada etapa** de transformação:

#### Etapa `{N}` — `{nome_da_etapa}`

| Item | Descrição |
|------|-----------|
| **Inputs** | `{tabelas ou DataFrames anteriores}` |
| **Tipo** | `Join` / `Aggregation` / `Window` / `Filter` / `Pivot` / `UDF` |
| **Chave (se join)** | `{lista_de_colunas}` |
| **Tipo de Join** | `inner` / `left` / `anti` / `semi` |
| **Regra de Negócio** | `{descrição em linguagem natural}` |
| **Pseudocódigo / Snippet** | ver abaixo |
| **Output** | `{nome_DF_resultado, colunas principais}` |
| **Observações** | `{skew conhecido, nulls esperados, etc.}` |

```python
# Pseudocódigo ou snippet real
df_resultado = (
    df_a
    .join(df_b, on=["customer_id", "event_dt"], how="left")
    .filter(F.col("status") == "ACTIVE")
    .groupBy("customer_id", "event_dt")
    .agg(
        F.sum("amount").alias("total_amount"),
        F.count("*").alias("tx_count"),
    )
)
```

### 6.3 Regras de Negócio

| ID | Regra | Origem (fonte do requisito) |
|----|-------|------------------------------|
| RN-01 | `{descrição da regra}` | `{ticket / reunião / owner}` |
| RN-02 | `{descrição da regra}` | `{...}` |

### 6.4 Tratamentos Especiais

- **Deduplicação:** `{estratégia, ex: row_number por chave ordenado por updated_at DESC}`
- **Late-arriving data:** `{janela de reprocessamento, ex: D-3}`
- **Nulls / Defaults:** `{regras por coluna}`
- **Skew:** `{se há chaves skewed, como é mitigado - salting, broadcast, etc.}`
- **Broadcast Joins:** `{quais tabelas são pequenas o suficiente para broadcast}`

---

## 7. Implementação PySpark

### 7.1 Estrutura do Código

```text
{repo_do_processo}/
├── src/
│   ├── {processo}/
│   │   ├── __init__.py
│   │   ├── main.py              # Entry point (chamado pela Step Function)
│   │   ├── config.py            # Config por ambiente
│   │   ├── readers/             # Leitura das origens
│   │   │   ├── hive_reader.py
│   │   │   └── iceberg_reader.py
│   │   ├── transformations/     # Lógica de negócio (pura, testável)
│   │   │   ├── joins.py
│   │   │   ├── aggregations.py
│   │   │   └── rules.py
│   │   ├── writers/             # Escrita Iceberg
│   │   │   └── iceberg_writer.py
│   │   └── quality/             # Data quality checks
│   │       └── expectations.py
│   └── tests/
│       ├── unit/
│       └── integration/
├── jobs/
│   └── run_job.py               # Wrapper executado no cluster
├── infra/                       # Terraform
└── pyproject.toml
```

### 7.2 Configuração do SparkSession

```python
spark = (
    SparkSession.builder
    .appName("{nome_do_job}")
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    .config("spark.sql.catalog.glue_catalog", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.glue_catalog.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog")
    .config("spark.sql.catalog.glue_catalog.warehouse", "s3://{bucket}/iceberg/")
    .config("spark.sql.catalog.glue_catalog.io-impl", "org.apache.iceberg.aws.s3.S3FileIO")
    .config("spark.sql.adaptive.enabled", "true")
    .config("spark.sql.adaptive.skewJoin.enabled", "true")
    .getOrCreate()
)
```

### 7.3 Parâmetros de Execução

| Parâmetro | Descrição | Default | Exemplo |
|-----------|-----------|---------|---------|
| `--env` | Ambiente (`dev`/`hml`/`prd`) | `dev` | `prd` |
| `--reference-date` | Data de referência do processamento | `{today-1}` | `2026-04-21` |
| `--backfill` | Reprocessamento histórico | `false` | `true` |
| `--start-snapshot-id` | Snapshot Iceberg de início (incremental) | N/A | `123456789` |

### 7.4 Configuração de Cluster (EMR / Glue)

| Ambiente | Tipo | Driver | Executors | Memória/Exec | Cores/Exec |
|----------|------|--------|-----------|--------------|------------|
| dev | EMR | m5.xlarge | 2 | 8g | 4 |
| hml | EMR | m5.2xlarge | 4 | 16g | 4 |
| prd | EMR | m5.4xlarge | 10-20 (autoscaling) | 32g | 8 |

### 7.5 Tuning Spark Aplicado

```properties
spark.sql.shuffle.partitions = {auto via AQE}
spark.sql.adaptive.enabled = true
spark.sql.adaptive.coalescePartitions.enabled = true
spark.sql.adaptive.skewJoin.enabled = true
spark.sql.autoBroadcastJoinThreshold = 100MB
spark.serializer = org.apache.spark.serializer.KryoSerializer
spark.sql.iceberg.vectorization.enabled = true
```

---

## 8. Orquestração — AWS Step Functions

### 8.1 State Machine

| Atributo | Valor |
|----------|-------|
| **Nome** | `sfn-{dominio}-{processo}-{env}` |
| **Tipo** | `STANDARD` / `EXPRESS` |
| **Trigger** | `EventBridge` / `Manual` / `Upstream SFN` |
| **Cron** | `{expressão cron}` |
| **Timeout** | `{minutos}` |

### 8.2 Diagrama de Estados

```text
[Start]
   │
   ▼
[ValidatePreconditions] ──fail──▶ [NotifyFailure]
   │
   ▼
[CheckUpstreamFreshness] ──stale──▶ [WaitAndRetry]
   │
   ▼
[StartEMRJob / GlueJob]
   │
   ▼
[WaitForCompletion]
   │
   ▼
[RunDataQualityChecks] ──fail──▶ [Quarantine + Alert]
   │
   ▼
[PublishDataContract / UpdateCatalog]
   │
   ▼
[NotifySuccess]
   │
   ▼
[End]
```

### 8.3 Estados Principais

| Estado | Tipo | Serviço | Descrição |
|--------|------|---------|-----------|
| `ValidatePreconditions` | Task | Lambda | Verifica parâmetros e dependências |
| `CheckUpstreamFreshness` | Task | Lambda / Athena | Confirma se origens estão atualizadas |
| `StartEMRJob` | Task | EMR Serverless | Dispara job PySpark |
| `RunDataQualityChecks` | Task | Glue / Lambda | Valida contratos e expectativas |
| `NotifyFailure` | Task | SNS | Publica alerta (Slack/email/PagerDuty) |

### 8.4 Retry & Error Handling

| Estado | MaxAttempts | Backoff | Errors Capturados |
|--------|-------------|---------|-------------------|
| `StartEMRJob` | 2 | Exponential (2x) | `States.TaskFailed` |
| `RunDataQualityChecks` | 1 | — | `QualityGateFailed` → quarentena |

### 8.5 Dependências Upstream / Downstream

| Tipo | State Machine / Processo | Gatilho |
|------|--------------------------|---------|
| Upstream | `{sfn-outro-processo}` | Deve terminar OK antes |
| Downstream | `{sfn-consumidor-a}` | Disparado via EventBridge ao sucesso |

---

## 9. Infraestrutura — Terraform

### 9.1 Estrutura dos Módulos

```text
infra/
├── modules/
│   ├── step-function/         # Máquina de estados
│   ├── emr-serverless/        # Aplicação EMR
│   ├── glue-catalog/          # Tabela Iceberg no Glue
│   ├── iam/                   # Roles e policies
│   ├── s3/                    # Buckets warehouse e logs
│   └── eventbridge/           # Schedules e triggers
└── environments/
    ├── dev/
    ├── hml/
    └── prd/
```

### 9.2 Recursos Provisionados

| Recurso | Terraform Resource | Descrição |
|---------|--------------------|-----------|
| State Machine | `aws_sfn_state_machine` | Orquestração do pipeline |
| EMR App | `aws_emrserverless_application` | Cluster de execução |
| IAM Role (exec) | `aws_iam_role` | Role do job (least privilege) |
| IAM Role (sfn) | `aws_iam_role` | Role da Step Function |
| Glue Table | `aws_glue_catalog_table` | Registro Iceberg no catálogo |
| S3 Bucket | `aws_s3_bucket` | Warehouse e logs |
| EventBridge Rule | `aws_cloudwatch_event_rule` | Schedule cron |
| CloudWatch Alarm | `aws_cloudwatch_metric_alarm` | Alerta de falha |
| SNS Topic | `aws_sns_topic` | Notificações |

### 9.3 Variáveis Principais

```hcl
variable "environment"       { type = string }
variable "domain"            { type = string }
variable "process_name"      { type = string }
variable "schedule_cron"     { type = string }
variable "emr_release_label" { type = string }
variable "driver_config"     { type = map(any) }
variable "executor_config"   { type = map(any) }
variable "notification_sns_arn" { type = string }
```

### 9.4 Backend & State

| Ambiente | Backend | Bucket | Key |
|----------|---------|--------|-----|
| dev | S3 | `tfstate-dev-{conta}` | `{dominio}/{processo}/terraform.tfstate` |
| hml | S3 | `tfstate-hml-{conta}` | `{dominio}/{processo}/terraform.tfstate` |
| prd | S3 | `tfstate-prd-{conta}` | `{dominio}/{processo}/terraform.tfstate` |

### 9.5 Pipeline de Deploy

```text
PR ──▶ terraform fmt + validate ──▶ tflint ──▶ checkov
                                                  │
                                                  ▼
                                    terraform plan (PR comment)
                                                  │
                                               merge main
                                                  │
                                                  ▼
                                    terraform apply (dev → hml → prd)
```

---

## 10. Data Quality & Observabilidade

### 10.1 Expectativas (Great Expectations / Deequ / Custom)

| ID | Regra | Severidade | Ação se falhar |
|----|-------|-----------|----------------|
| DQ-01 | `id` NOT NULL | CRÍTICA | Aborta o pipeline |
| DQ-02 | `event_dt` entre `{inicio}` e `{hoje}` | ALTA | Aborta o pipeline |
| DQ-03 | `customer_id` distinct count ≥ `{N}` | MÉDIA | Alerta, não aborta |
| DQ-04 | Volumetria diária dentro de ±`{X%}` da média 7d | MÉDIA | Alerta |
| DQ-05 | `amount` ≥ 0 | ALTA | Quarantine das linhas |

### 10.2 Métricas Emitidas

| Métrica | Fonte | Destino |
|---------|-------|---------|
| Linhas lidas por fonte | PySpark accumulator | CloudWatch |
| Linhas escritas | Iceberg snapshot summary | CloudWatch |
| Duração do job | Step Function | CloudWatch |
| Tempo de SLA | Step Function | CloudWatch |
| Custo estimado | Cost Explorer tags | Dashboard FinOps |

### 10.3 Logging

- **Driver/Executors:** CloudWatch Logs em `/aws/emr-serverless/{app_id}`
- **Step Function:** CloudWatch Logs em `/aws/vendedlogs/states/{sfn_name}`
- **Formato:** JSON estruturado com `trace_id`, `process`, `env`, `reference_date`

### 10.4 Alertas

| Evento | Canal | Destinatário |
|--------|-------|-------------|
| Falha do job | Slack `#alerts-{dominio}` | On-call + owner |
| SLA breach (> `{X}` min) | PagerDuty | On-call |
| Data quality crítica | Slack + email | Owner + stakeholders |

---

## 11. Segurança & Governança

| Item | Detalhe |
|------|---------|
| **Classificação dos dados** | `Público` / `Interno` / `Restrito` / `Confidencial` |
| **Campos PII** | `{lista}` |
| **LGPD** | Base legal: `{ex: legítimo interesse}` |
| **Mascaramento** | `{estratégia para não-prod}` |
| **Criptografia at rest** | S3 SSE-KMS (`{alias_kms}`) |
| **Criptografia in transit** | TLS 1.2+ |
| **Controle de acesso** | Lake Formation tags / IAM / bucket policy |
| **Audit trail** | CloudTrail + Lake Formation audit |

### 11.1 Roles IAM

| Role | Usada por | Permissões mínimas |
|------|-----------|--------------------|
| `role-sfn-{processo}` | Step Function | `emr:StartJobRun`, `states:*`, `sns:Publish` |
| `role-emr-{processo}` | Job EMR | `s3:Get/Put` (paths específicos), `glue:Get*`, `kms:Decrypt` |

---

## 12. Runbook / Troubleshooting

### 12.1 Cenários Comuns

| Sintoma | Causa Provável | Diagnóstico | Ação |
|---------|---------------|-------------|------|
| Job falha com `OutOfMemory` | Skew em join / partição grande | Ver Spark UI → Stages → Tasks | Aumentar executor memory ou aplicar salting |
| `ConcurrentModificationException` no Iceberg | Escritas concorrentes | Check snapshot history | Retry ou revisar orquestração |
| Volumetria fora do esperado | Atraso na fonte | Query freshness upstream | Re-run após upstream |
| `NoSuchNamespaceException` | Catalog mal configurado | Verificar conf do Spark | Ajustar `spark.sql.catalog.*` |
| Job lento (> SLA) | Shuffle excessivo | Spark UI → SQL tab | Reavaliar partições / broadcast |

### 12.2 Reprocessamento (Backfill)

```bash
# Reprocessar data específica
aws stepfunctions start-execution \
  --state-machine-arn {arn} \
  --input '{
    "reference_date": "2026-04-15",
    "backfill": true
  }'

# Reprocessar janela
for dt in $(seq ...); do
  aws stepfunctions start-execution --input "..."
done
```

### 12.3 Rollback

| Situação | Procedimento |
|----------|--------------|
| Dados ruins publicados | `CALL system.rollback_to_snapshot('{tbl}', {snapshot_id})` |
| Código com bug | Rollback via tag Git + novo deploy |
| Infra quebrada | `terraform apply` de commit anterior |

---

## 13. Testes

| Tipo | Framework | Cobertura mínima |
|------|-----------|------------------|
| Unitário (transformações puras) | pytest + chispa | 80% |
| Integração (local Spark) | pytest + testcontainers | Principais joins |
| Contrato de dados | Great Expectations | 100% campos críticos |
| E2E (ambiente dev) | Step Function exec + assert na Iceberg | Smoke |

---

## 14. Custos & Capacity

| Item | Estimativa Mensal |
|------|-------------------|
| EMR Serverless (compute) | `$ {valor}` |
| S3 (storage + requests) | `$ {valor}` |
| Glue Catalog | `$ {valor}` |
| Step Functions | `$ {valor}` |
| CloudWatch Logs | `$ {valor}` |
| **Total estimado** | `$ {valor}` |

**Tags obrigatórias:** `domain`, `data-product`, `environment`, `cost-center`, `owner`.

---

## 15. Referências

- Código: `{repo_url}`
- Pipeline CI/CD: `{url}`
- Dashboard de monitoramento: `{url_grafana/cloudwatch}`
- Data Catalog: `{url_datahub/glue}`
- Data Contract: `{url}`
- Ticket/Epic: `{url_jira}`
- Reuniões de definição: `{links}`

---

## 16. Histórico de Alterações

| Data | Versão | Autor | Mudança |
|------|--------|-------|---------|
| YYYY-MM-DD | 1.0.0 | `{nome}` | Criação inicial |
| YYYY-MM-DD | 1.1.0 | `{nome}` | `{descrição}` |

---

## 17. Checklist de Aprovação

Antes de ir para produção:

- [ ] Requisitos validados com stakeholders
- [ ] Data contract publicado e aprovado
- [ ] Terraform revisado (2 approvers)
- [ ] Testes unitários ≥ 80% cobertura
- [ ] Teste E2E executado em HML com dados representativos
- [ ] Data quality checks configurados e testados (incluindo falhas)
- [ ] Alertas configurados e testados
- [ ] Runbook revisado pelo on-call
- [ ] Custo estimado aprovado por FinOps
- [ ] Classificação de dados e mascaramento validados por Segurança/Privacidade
- [ ] Catálogo atualizado (discoverability)
- [ ] Treinamento / handover para time de suporte realizado
