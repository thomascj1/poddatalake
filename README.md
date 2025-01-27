# poddatalake
Projeto de Datalake da PoD Academy para engenharia de dados jr

# Projeto Data Lake - PoD Cartões

Bem-vindo(a) ao repositório do projeto **Data Lake - PoD Cartões**. Este documento descreve o objetivo do projeto, suas dependências, a arquitetura proposta, os serviços utilizados, os dados envolvidos e, por fim, o diagrama de fluxo de processamento (DAG).

---

## 1. Introdução

O objetivo deste projeto é estabelecer uma arquitetura de Data Lake escalável e governada, que permita à PoD Cartões:

- **Centralizar** dados de cartões de crédito (faturas, transações, pagamentos, etc.).
- **Preparar** dados para análises avançadas e criação de modelos de crédito, fraude e inadimplência.
- **Fornecer** um Book de Variáveis de Fatura (U1M, U3M, U6M, U12M) a ser utilizado pelos cientistas de dados.
- **Garantir** qualidade de dados, segurança e governança ao longo de todo o pipeline.

Neste repositório, você encontrará **scripts**, **documentação** e **diagramas** sobre como tudo está organizado e orquestrado.

---

## 2. Dependências

Para executar ou reproduzir este projeto, você precisa ter algumas ferramentas instaladas/configuradas:

1. **Python 3.x** (para scripts e para o Airflow, caso seja necessário em ambiente local)
2. **Apache Airflow** ou **MWAA (Managed Workflows for Apache Airflow)** (para orquestração)
3. **AWS CLI** (para interação com serviços AWS)
4. **Serviços AWS**:
   - Amazon S3
   - AWS Glue
   - Amazon EMR (ou AWS Glue Jobs/Spark)
   - Amazon Athena
   - Amazon Redshift (opcional, dependendo do consumo)
   - Amazon SageMaker (opcional, caso haja utilização para ML)
5. **Conta AWS** com permissões para criar e gerenciar os recursos acima (S3, Glue Catalog, etc.)

**Opcional**: Dependências específicas de **Data Quality** (bibliotecas Python), ou frameworks de Big Data (Spark). Consulte o arquivo `requirements.txt` (caso exista) para mais detalhes.

---

## 3. Diagrama da Arquitetura

Abaixo apresentamos a visão macro da arquitetura de dados no ecossistema AWS:

