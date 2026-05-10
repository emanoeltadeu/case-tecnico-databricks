# 🚀 Case Técnico: Engenharia de Dados (Arquitetura Medallion)

[![Notebook Quality (Linter)](https://github.com/emanoeltadeu/case-tecnico-databricks/actions/workflows/notebook_quality.yml/badge.svg)](https://github.com/emanoeltadeu/case-tecnico-databricks/actions/workflows/notebook_quality.yml)
![Spark](https://img.shields.io/badge/Spark-3.4%2B-orange?logo=apachespark)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-Blue?logo=delta)
![Databricks](https://img.shields.io/badge/Databricks-Community-red?logo=databricks)

Este repositório apresenta uma solução completa de Engenharia de Dados utilizando a arquitetura **Medallion (Bronze, Silver, Gold)** no Databricks. O projeto transforma fontes de dados heterogêneas e inconsistentes em um modelo analítico estruturado (**Star Schema**) otimizado para tomada de decisão.

---

## 🏗️ Arquitetura da Solução

A solução segue o fluxo de dados em três camadas lógicas sobre o **Delta Lake**, garantindo transações ACID e performance:

1.  **🥉 Camada Bronze (Raw)**: Ingestão fiel à origem (CSV, JSON, XLSX, TXT) com metadados de linhagem.
    *   *Destaque*: Implementação de carga incremental via **Watermark** para otimização de processamento.
2.  **🥈 Camada Silver (Refined)**: Limpeza profunda, padronização e **Data Quality Gate**.
    *   *Destaque*: Uso intensivo de **Regex** para sanear campos e criação de flags de qualidade (`flg_qualidade`) para auditoria.
3.  **🥇 Camada Gold (Curated)**: Modelagem dimensional (**Star Schema**) voltada para performance de BI.
    *   *Destaque*: Dimensões consolidadas e cálculo preciso de KPIs (Receita Líquida com rateio de desconto e SLA Logístico).

---

## 📁 Estrutura do Projeto

```bash
├── .github/workflows/    # CI/CD: Linter automático de qualidade de código
├── docs/                 # Documentação detalhada e Resumo Executivo
├── notebooks/            # Código-fonte Spark (PySpark & SQL)
│   ├── utils             # Funções globais (Upsert, Optimize, Save)
│   ├── data_quality      # Funções de limpeza e tratamento de tipos
│   ├── 01_Bronze         # Pipeline de Ingestão
│   ├── 02_Silver         # Pipeline de Refinamento e Qualidade
│   ├── 03_Gold           # Modelagem Star Schema
│   └── 04_Insights       # KPIs e Respostas de Negócio
└── sources/              # Fontes brutas originais (CSV, JSON, etc)
```

---

## 🚀 Como Executar

1.  Importe os notebooks para o seu workspace Databricks.
2.  Configure a variável `catalog_name` no notebook `utils` ou no setup inicial.
3.  Execute os notebooks na ordem numérica: `01` -> `02` -> `03` -> `04`.
4.  O notebook **`04_Business_Insights`** contém as respostas visuais para os desafios de negócio do case.

---

## 📊 Governança e Qualidade

O projeto vai além do código Spark, incorporando boas práticas de engenharia de software:
- **CI/CD**: GitHub Action que valida automaticamente a sintaxe e o estilo do código (PEP8) em todos os notebooks.
- **Data Dictionary**: Documentação completa de cada campo e sua transformação (veja em `/docs`).
- **Resumo Executivo**: Visão estratégica sobre as decisões técnicas tomadas.

---

## 🛠️ Tecnologias
- **Linguagens**: PySpark, Spark SQL, Python.
- **Armazenamento**: Delta Lake (Acid, Time Travel, Z-Order).
- **Ambiente**: Databricks Community Edition.
- **Engenharia**: GitHub Actions, Git Flow (PR-based).