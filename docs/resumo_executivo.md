# Resumo Executivo Técnico - Pipeline de Dados (Bronze-Silver-Gold)

## 1. Visão Geral da Solução
A solução foi estruturada utilizando a **Arquitetura Medallion** (Bronze, Silver e Gold) no Databricks, focando em robustez, escalabilidade e, principalmente, facilidade de consumo para o Analista de BI.

### Fluxo de Dados:
- **Bronze (Raw)**: Ingestão de fontes heterogêneas (CSV, JSON, Excel) mantendo a fidelidade total aos dados de origem.
- **Silver (Cleaned/Refined)**: Camada de normalização, tratamento de nulos, imposição de esquemas e validação de qualidade.
- **Gold (Curated)**: Camada analítica modelada para performance e clareza (Star Schema).

---

## 2. Principais Decisões Técnicas

### 2.1. Arquitetura Enxuta (Modularização vs. Complexidade)
Optamos por uma estrutura de notebooks agrupados por camada lógica (`01_Ingestion`, `02_Refining`, `03_Curated`) em vez de notebooks individuais por tabela.
- **Justificativa**: Melhora a rastreabilidade do pipeline, reduz o overhead de orquestração no Databricks Community e facilita a revisão do código por outros engenheiros.

### 2.2. Centralização de Data Quality
Criamos um notebook exclusivo (`data_quality.ipynb`) que centraliza as "regras do jogo".
- **Justificativa**: Garante que uma premissa de negócio (ex: "nulos viram 'NAO INFORMADO'") seja aplicada uniformemente em todo o projeto.

### 2.3. Schema Enforcement (Imposição de Tipos)
Utilizamos `StructType` do PySpark para forçar a tipagem na camada Silver.
- **Justificativa**: Blindagem contra mudanças de esquema na fonte e performance superior em comparação ao uso de bibliotecas de validação de software (como Pydantic) em ambiente distribuído.

### 2.4. Otimização com Z-Order e Delta Lake
Implementamos o comando `OPTIMIZE` com `ZORDER` nas dimensões principais.
- **Justificativa**: Otimiza fisicamente a disposição dos dados para acelerar os filtros e joins realizados pelo Analista de BI.

### 2.5. Estratégia de Qualidade de Dados (Silver Gate)
Implementamos uma abordagem de "Ingestão Raw na Bronze" e "Validação Lógica na Silver".
- **Justificativa**: A camada Bronze mantém a fidelidade total à origem para fins de auditoria técnica. A inteligência de negócio reside na Silver, onde registros que violam regras críticas são sinalizados (`flg_qualidade = False`) em vez de excluídos. Isso permite que o Analista de BI tenha visibilidade total sobre o "lixo" que entra e possa filtrar os dados limpos sem perder o histórico do que foi rejeitado.
- **Exemplos de Regras Aplicadas**:
    - **Detecção de Duplicidade Interna**: No arquivo de itens de pedido (CSV), identificamos casos onde o mesmo `product_code` aparece repetido para o mesmo `order_id` com sequenciais diferentes (ex: Pedido `O00033`). Usamos *Window Functions* para marcar esses registros.
    - **Integridade Financeira**: Validamos se o Valor Líquido é menor ou igual ao Valor Bruto. Se o líquido for maior, o registro é marcado como inconsistente.
    - **Tratamento de Malformação Numérica**: Implementamos validação via **Expressão Regular (Regex)** para capturar valores não numéricos em campos de preço (ex: o valor `'N/A'` encontrado no JSON de produtos), convertendo-os para `NULL` de forma segura.

### 2.6. Processamento Incremental (Watermark)
Implementamos lógica de carga incremental utilizando o conceito de *Watermark* para as tabelas de fatos e grandes dimensões (Produtos e Pedidos).
- **Justificativa**: Em vez de reprocessar toda a base a cada execução, o pipeline consulta a última data de atualização processada (`updated_at` ou `data_atualizacao`) na Silver e busca na Bronze apenas os registros mais recentes. Isso reduz drasticamente o consumo de recursos e o tempo de execução do pipeline.

---

## 3. Modelagem Analítica Final (Gold)

A camada Gold foi desenhada seguindo o padrão **Star Schema** (Fatos e Dimensões), focada exclusivamente na usabilidade para o Analista de BI. 

### 3.1. Estratégia de Modelagem
Em vez de tabelas únicas e gigantescas ("tabelões"), optamos pela separação clara de entidades e eventos para evitar o **fan-out** (explosão de registros em joins) que distorce métricas agregadas.

- **Dimensões (O Contexto)**:
    - `dim_clientes`: Cadastro único de clientes.
    - `dim_produtos`: Cadastro de produtos com categorias e subcategorias.
    - `dim_vendedores`: **Consolidação Denormalizada**. Unificamos as tabelas de Vendedores, Canais de Venda e Regiões. Isso permite que o analista filtre "Região" ou "Tipo de Canal" diretamente na dimensão do vendedor, sem joins adicionais.
- **Fatos (As Métricas)**:
    - `fact_vendas`: Granularidade no nível de **Item de Pedido**. Inclui o rateio proporcional do desconto do cabeçalho para cada item, permitindo o cálculo exato da **Receita Líquida** e **Ticket Médio**.
    - `fact_entregas`: Granularidade no nível de **Pedido/Entrega**. Focada em performance logística (SLA), calcula automaticamente os `dias_atraso` e a `flg_atraso` comparando a data real de entrega com a data prometida.

### 3.2. Granularidade e Relacionamentos
- A `fact_vendas` conecta-se com `dim_clientes`, `dim_produtos` e `dim_vendedores`.
- A `fact_entregas` conecta-se com `dim_vendedores` (via pedido) e `dim_clientes`.
- Essa estrutura garante que um Analista de BI consiga cruzar, por exemplo, a "Taxa de Atraso por Categoria de Produto" de forma performática e correta.

---

## 4. Desafios Encontrados e Superados
- **Leitura de Excel no Databricks**: Superamos a ausência de bibliotecas Maven no ambiente Community utilizando uma rotina personalizada de leitura via Pandas integrada ao Spark.
- **Inconsistência de Fontes**: Tratamos variações complexas de data e geografia (UFs) através de uma camada de normalização defensiva na Silver.
- **Erros de Casting no Spark 3.4+**: Resolvemos falhas de conversão de decimais em strings (ex: '3.0' para INT) criando funções utilitárias personalizadas (`clean_int`, `clean_numeric`) no `data_quality.ipynb`.

---

## 5. Próximos Passos Recomendados (Evolução da Solução)

A arquitetura atual atende perfeitamente aos requisitos do case, mas para um cenário produtivo de larga escala, recomendamos as seguintes evoluções:

- **Observabilidade e Qualidade de Dados (Soda Core)**: Substituir (ou complementar) as flags booleanas estáticas (`flg_qualidade`) por uma suíte de testes robusta utilizando o **Soda (Python)**. O Soda permite escrever contratos de dados em YAML, executar testes nativos contra as tabelas Delta e gerar relatórios automatizados de *Data Observability*, parando o pipeline caso regras críticas sejam violadas (Circuit Breaker).
- **Orquestração Avançada**: Migrar a execução sequencial (feita via comandos mágicos `%run`) manual para o **Databricks Workflows** ou ferramentas externas como **Apache Airflow**. Isso garantirá gestão de dependências via DAGs, retentativas automáticas e alertas de falha.
- **CI/CD e Repositórios**: Integrar Github Action com Databricks. Quando um Pull Requests de um notebook for aprovado, ir para o ambiente de workspace em produção.
- **Tratamento de Anomalias (Quarentena)**: Implementar limites máximos de sanidade (thresholds) para campos numéricos críticos. Isso evita que erros sistêmicos (como um código de barras lido como `quantidade`) poluam a base. Registros absurdos devem ser desviados automaticamente para uma **tabela de quarentena**, permitindo investigação manual sem interromper o fluxo do pipeline principal.

---

## 6. Cultura de Engenharia (DevOps / CI-CD)

Este projeto não foi apenas focado em processar dados, mas em como o código é mantido:
- **Linting Automático**: Implementamos uma GitHub Action que valida a sintaxe Python e o estilo PEP8 nos notebooks a cada Pull Request. Isso garante que o código seja legível e livre de erros básicos de sintaxe antes de chegar à branch principal.
- **Governança de Branch**: Configuração de proteção de branch (`main`) exigindo Pull Requests, simulando um ambiente real de revisão de código por pares (Code Review).
