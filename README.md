1. Objetivo e Contexto

A PoD Cartões deseja criar um Data Lake para armazenar e processar dois arquivos mensais:

tb_faturas
tb_pagamentos

Esses arquivos serão disponibilizados pelo cliente todo mês e enviados para a pasta de ingestão em um bucket S3 (camada Raw). O objetivo é:

- Centralizar os dados de faturas e pagamentos.
- Garantir qualidade e governança das informações.
- Preparar indicadores e métricas (Book de Variáveis de Fatura) para análises de inadimplência, comportamento de pagamento e risco de crédito.
- Otimizar custos e prever gastos na nuvem, incluindo alertas de custos na AWS.
- Facilitar o consumo de dados via Athena, QuickSight ou outras ferramentas de análise

  
2. Arquitetura de Dados

A arquitetura segue o padrão de camadas no Amazon S3, usando AWS Glue para o processamento (ETL) e Airflow (ou MWAA - Managed Workflows for Apache Airflow) para orquestração. Há três camadas no S3:

Raw:
Armazena dados brutos, sem qualquer modificação.
Recebe mensalmente os arquivos tb_faturas e tb_pagamentos.

Curated:
Concentra dados limpos e padronizados após validações, deduplicação e aplicação de schemas.
Dados são convertidos, geralmente, para Parquet (particionado), para facilitar queries e melhorar performance.

Refined:
Camada com dados já agregados e transformados de forma analítica.
É onde se encontra o Book de Variáveis de Fatura, pronto para consumo via Athena, QuickSight ou outros serviços.

Diagrama Geral

                   +-----------------------+
                   |  Arquivos Mensais    |
                   |  (tb_faturas,        |
                   |   tb_pagamentos)     |
                   +----------+-----------+
                              |
                              v
          +-------------------------------------------------+
          |            S3 (Camada Raw)                      |
          +-----------------+-------------------------------+
                            |
                            v
            +-------------------------------------+
            | Orquestração (Airflow ou MWAA)      |
            +-----------------+--------------------+
                            |
                            v
                    +---------------+
                    |  AWS Glue     |  <--- ETL (limpeza, deduplicação)
                    |  (Jobs)       |
                    +-------+-------+
                            |
                            v
          +-------------------------------------------------+
          |          S3 (Camada Curated)                    |
          +-----------------+-------------------------------+
                            |
                            v
                    +---------------+
                    |  AWS Glue     |  <--- Agregações e Cálculos
                    |  (Jobs)       |       (Book de Variáveis)
                    +-------+-------+
                            |
                            v
          +-------------------------------------------------+
          |         S3 (Camada Refined)                     |
          +-----------------+-------------------------------+
                            |
                            v
              +-------------------+    +---------------------+
              | Amazon Athena     |    | Amazon QuickSight    |
              | (Consulta SQL)    |    | (Dashboards/BI)      |
              +-------------------+    +---------------------+

Observações:

O AWS Glue também se encarrega de manter um Data Catalog com metadados (schema, partições).
A governança e segurança podem ser gerenciadas com AWS IAM e, se necessário, AWS Lake Formation.
3. Orquestração e Fluxo de Processamento

Visão (DAG do Airflow)
Uma DAG (Directed Acyclic Graph) no Airflow coordena as etapas:

Descrição dos Passos:

INSERIR DAG AIRFLOW

Sensor para Novos Arquivos:
Airflow verifica na pasta s3://<bucket>/raw/ingestion/ se existem os arquivos tb_faturas_YYYYMM.csv e tb_pagamentos_YYYYMM.csv.
Ingestão para S3 (Raw):
Se os arquivos chegam por outro meio (FTP, e-mail, etc.), a DAG pode movê-los ou copiá-los para a pasta de ingestão, garantindo que fiquem armazenados como “fonte da verdade”.
Job Glue - Processamento Curated:
Aplica regras de limpeza, validação de schema, deduplicação.
Converte para Parquet e gera tabelas/partições na camada Curated.
Job Glue - Book de Variáveis:
Realiza agregações e cálculos específicos.
Gera a visão final (Refined), particionada por data ou por referência (ex.: dt_base=2024-02-01).
Notificação:
Em caso de sucesso, envia mensagem confirmando a execução.
Em caso de falha, aciona planos de contingência (descritos na Seção 7).

4. Data Quality

Para assegurar a integridade dos dados:

Validação de Schema:
Verificar se tb_faturas e tb_pagamentos vêm com a quantidade de colunas esperadas, tipos corretos, etc.
Tipagem:
Em Glue, definir se colunas são STRING, INT, DECIMAL, DATE, etc.
Lidar com registros corrompidos ou inválidos (enviar para “quarentena” ou descartar).
Deduplicação:
Usar chaves de unicidade (ex.: id_fatura, id_pagamento).
Rotinas do Glue ou Spark para remover linhas duplicadas.
Particionamento:
Normalmente, particionamos por ano=YYYY/mes=MM ou dt_base=YYYY-MM-DD.
Melhora performance e reduz custo de queries.
Reconciliação:
Verificar se o número de registros na camada Curated corresponde ao da Raw (exceto os removidos por falha de schema ou duplicidade).
Logar quantos foram excluídos ou corrigidos.
5. Book de Variáveis de Fatura

5.1. Conceito e Visões
O Book de Variáveis consolida informações de faturas e pagamentos em janelas temporais (U1M, U3M, U6M, U12M) a partir de uma ref_date (por exemplo, 2024-02-01). Cada visão reflete quantas faturas e pagamentos ocorreram nesse período, qual o valor total, qual o percentual de pagamento em dia ou de inadimplência, etc.

5.2. Quantidade de Variáveis
No exemplo de código SQL, são produzidas cerca de 20 colunas. Principais métricas:

total_faturas, valor_total_faturas, valor_medio_faturas
total_pagamentos, valor_total_pagamentos, valor_medio_pagamentos
percentual_pagamento_prazo, percentual_inadimplencia
total_atrasos, percentual_atrasos, media_dias_atraso, max_dias_atraso
valor_inadimplente, percentual_valor_pago
total_clientes, media_fatura_por_cliente, media_pagamento_por_cliente
E outras colunas auxiliares como periodo.

6. Alertas de Custos na AWS

Para evitar surpresas na fatura, recomenda-se:

AWS Budgets
Definir um budget (ex.: “X dólares por mês”) e receber alertas via e-mail/SNS quando se atingir 80%, 90% ou 100% do budget.
Billing Alerts via CloudWatch
Criar alarmes no CloudWatch com a métrica “EstimatedCharges” (USD) para a conta.
Enviar notificação (SNS, e-mail) quando o valor ultrapassar um limite definido.
Acompanhamento no Cost Explorer
Ferramenta visual que permite analisar tendências de custo diário e prever o fechamento do mês.
Facilita identificar serviços que estão gerando mais gastos (ex.: S3, Glue, QuickSight).
Essas medidas proporcionam maior governança e tranquilidade para o time, principalmente em projetos de Data Lake em expansão.

7. Planos de Contingência

Falha no Job Glue
Logs no CloudWatch para verificar o erro.
Tentativas automáticas (retry) definidas no Airflow.
Se persistir, isolar o problema e reexecutar somente o subset de dados afetado.
Rollback e Versionamento
Habilitar o Versioning no bucket S3, possibilitando a restauração de arquivos de camadas Curated/Refined em caso de gravações indevidas.
Interromper a DAG antes de popular a Refined, se houver inconsistências.
Detecção de Arquivos
Se o arquivo mensal não aparecer no tempo previsto, Airflow alerta o time responsável (via e-mail/Slack) para verificar com o cliente.
Erros de Schema
Se a estrutura do CSV vier fora do padrão, direcionar para uma “zona de quarentena” no S3, notificando o time para corrigir manualmente.
8. Conclusão e Próximos Passos

Conclusão
Com esse projeto:

A PoD Cartões passa a ter um Data Lake centralizado, governado e escalável no ecossistema AWS.
A ingestão de dois arquivos mensais (tb_faturas e tb_pagamentos) torna-se automatizada, com etapas claras de validação, limpeza e geração de métricas.
O Book de Variáveis de Fatura (U1M, U3M, U6M, U12M) fornece insights para modelos de crédito, inadimplência e comportamento de pagamento.
Alertas de custos garantem previsibilidade financeira e evitam estouros orçamentários.
