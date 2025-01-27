# Data Lake - PoD Cartões

Este repositório documenta o projeto de **Data Lake** desenvolvido para a **PoD Cartões**, cujo objetivo é processar e analisar dados de **faturas** e **pagamentos** de cartão de crédito. A iniciativa prevê a ingestão de dois arquivos mensais e a criação de um **Book de Variáveis de Fatura** para análises e modelos preditivos.

---

## 1. Objetivo do Projeto

1. **Centralizar** dados de faturas e pagamentos em um Data Lake escalável.  
2. **Garantir** a qualidade e a consistência dos dados, por meio de camadas (Raw, Curated, Refined).  
3. **Facilitar** a geração de insights e análises avançadas sobre inadimplência e comportamento de pagamento.  
4. **Criar** um Book de Variáveis de Fatura (janelas U1M, U3M, U6M, U12M) que sirva de base para modelos de crédito, risco e relatórios.  
5. **Monitorar** custos na AWS, com alertas e melhores práticas de governança.

---

## 2. Escopo e Contexto

- **Arquivos de Ingestão**:
  - `tb_faturas` (informações de fatura, valor, vencimento etc.)
  - `tb_pagamentos` (informações de data de pagamento, valor, atraso etc.)
- **Frequência**: Mensal (2 arquivos recebidos todo mês).
- **Serviços Principais**:
  - **Amazon S3** (para armazenamento em camadas)
  - **AWS Glue** (para ETL e catalogação de dados)
  - **Airflow** ou **MWAA** (para orquestração)
  - **Amazon Athena / QuickSight** (para análise e visualização)

---

## 3. Arquitetura de Dados

A solução adota **três camadas** no Amazon S3:

1. **Raw**  
   - Armazena dados **brutos**, sem modificações.  
   - Repositório de ingestão para `tb_faturas` e `tb_pagamentos` recebidos mensalmente.

2. **Curated**  
   - Concentra dados **limpos e padronizados** (deduplicação, validação de schema).  
   - Uso de **AWS Glue** para processar e converter para formato **Parquet**, com partições por mês/ano.

3. **Refined**  
   - Contém dados **agregados** e **transformados** para consumo analítico.  
   - Armazena principalmente o **Book de Variáveis de Fatura**, consolidado em janelas de tempo (U1M, U3M, U6M, U12M).

### Diagrama Simplificado

               +-----------------------+
               |  Arquivos Mensais    |
               | (tb_faturas,         |
               |  tb_pagamentos)      |
               +----------+-----------+
                          |
                          v
      +-------------------------------------------------+
      |            S3 (Raw)                             |
      | s3://<bucket>/raw/...                           |
      +-----------------+-------------------------------+
                        |
                        v
        +-------------------------------------+
        | Orquestração (Airflow ou MWAA)      |
        +-----------------+--------------------+
                        |
                        v
                +---------------+
                | AWS Glue      |  <--- ETL
                | (Jobs)        |
                +-------+-------+
                        |
                        v
      +-------------------------------------------------+
      |          S3 (Curated)                           |
      | s3://<bucket>/curated/...                       |
      +-----------------+-------------------------------+
                        |
                        v
                +---------------+
                | AWS Glue      |  <--- Agregações e
                | (Jobs)        |       métricas (Book)
                +-------+-------+
                        |
                        v
      +-------------------------------------------------+
      |         S3 (Refined)                            |
      | s3://<bucket>/refined/...                       |
      +-----------------+-------------------------------+
                        |
                        v
          +-------------------+    +---------------------+
          | Amazon Athena     |    | Amazon QuickSight    |
          +-------------------+    +---------------------+

          
---

## 4. Orquestração e Fluxo de Processamento

O pipeline é controlado por **Airflow** (ou MWAA). A **DAG** segue os seguintes passos:

1. **Detecção de Arquivos**
   - Verifica se `tb_faturas_YYYYMM.csv` e `tb_pagamentos_YYYYMM.csv` estão na pasta de ingestão.

2. **Ingestão para Raw**
   - Caso os arquivos estejam em outro local, copia-os para `s3://<bucket>/raw/ingestion`.

3. **Job Glue - Curated**
   - Limpeza, validação de schema, conversão para Parquet.
   - Saída: `s3://<bucket>/curated/...`

4. **Job Glue - Refined (Book de Variáveis)**
   - Cálculos e agregações para gerar indicadores (U1M, U3M, U6M, U12M).
   - Saída: `s3://<bucket>/refined/...`

5. **Notificações**
   - Alerta de sucesso ou falha via e-mail/Slack.

---

## 5. Data Quality e Governança

- **Validação de Schema**: Verificar se colunas e tipos batem com o esperado (ex.: 10 colunas em `tb_faturas`).
- **Deduplicação**: Usar chaves primárias (ex.: `id_fatura`, `id_pagamento`) para remover registros repetidos.
- **Particionamento**: Organizar dados por `ano=YYYY/mes=MM`, otimizando consultas.
- **Reconciliação**: Conferir contagem de registros entre Raw e Curated (excluindo descartados).
- **Governança**:
  - **IAM** para controlar quem acessa cada camada do S3.
  - **Possibilidade** de usar AWS Lake Formation para políticas avançadas de dados.

---

## 6. Book de Variáveis de Fatura

### Objetivo
Consolidar métricas de uso e pagamento de faturas, permitindo entender o comportamento do cliente em diferentes horizontes de tempo:
- **U1M** (último 1 mês)
- **U3M** (últimos 3 meses)
- **U6M** (últimos 6 meses)
- **U12M** (últimos 12 meses)

### Indicadores Principais
1. **Valores de Fatura**: total, média por cliente, atraso etc.
2. **Pagamento em Dia**: percentual de faturas pagas até a data de vencimento.
3. **Inadimplência**: percentual de faturas pagas abaixo do valor mínimo.
4. **Atrasos**: média e máximo de dias em atraso.
5. **Razão entre Pagamento e Fatura** (percentual_valor_pago).

Ao todo, obtemos cerca de **20 variáveis**, consultáveis via **Athena** ou utilizadas em ferramentas de BI/Ciência de Dados.

---

## 7. Alertas de Custos na AWS

Para evitar surpresas na fatura da AWS, recomenda-se:

1. **AWS Budgets**
   - Definir orçamentos mensais e alertar o time ao atingir certo percentual do budget.
2. **CloudWatch Alarms**
   - Criar alarmes baseados na métrica de `EstimatedCharges`, enviando notificação via SNS ou e-mail.
3. **Cost Explorer**
   - Revisar gastos diários para identificar picos de consumo e possíveis otimizações.

---

## 8. Planos de Contingência

1. **Falha na Detecção de Arquivos**
   - Retry programado; se persistir, notificar time responsável.
2. **Erro de Schema ou Deduplicação**
   - Registros fora do padrão podem ser enviados para uma “zona de quarentena” no S3.
   - Logs no CloudWatch para investigar.
3. **Rollback**
   - Manter Versioning no S3, permitindo restaurar dados caso um job gere resultados incorretos.
4. **Retries Automáticos**
   - Parametrizar Airflow para reexecutar tarefas em caso de falhas transitórias.

---

## 9. Conclusão

A presente solução de Data Lake permite à **PoD Cartões**:

- **Centralizar** dados de faturas e pagamentos em um repositório seguro e escalável.
- **Garantir** a qualidade dos dados por meio de validações e deduplicações no Glue.
- **Gerar** insights sobre inadimplência e comportamento de pagamento (Book de Variáveis).
- **Monitorar** gastos na AWS, evitando estouros orçamentários.
- **Facilitar** análises via Athena, QuickSight e eventuais modelos de Machine Learning.

### Próximos Passos
- **Conectar** QuickSight para criar dashboards gerenciais de inadimplência.
- **Implementar** modelos preditivos de risco de crédito.
- **Ampliar** escopo para novas fontes (ex.: histórico de negociações).

---

