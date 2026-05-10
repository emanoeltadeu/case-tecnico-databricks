# Documentação Técnica - Pipeline de Dados Medallion

Este documento detalha as decisões técnicas, regras de negócio e processos de engenharia aplicados no desenvolvimento da solução de dados.

## 1. Visão Geral da Arquitetura

A solução utiliza a **Arquitetura Medallion** sobre o **Delta Lake**, garantindo transações ACID e alta performance.

- **Bronze**: Ingestão bruta com preservação de linhagem.
- **Silver**: Camada de verdade técnica. Onde o dado é limpo, tipado e validado.
- **Gold**: Camada de verdade de negócio. Modelagem dimensional (Star Schema) pronta para consumo por BI.

## 2. Regras de Negócio e Transformações

### 2.1. Cálculo de Receita Líquida (Gold)
- **Desafio**: O desconto é informado apenas no cabeçalho do pedido (`pedidos`), mas a análise é feita por item (`pedidos_itens`).
- **Regra**: Aplicamos um rateio proporcional do desconto baseado no peso do valor bruto de cada item sobre o total do pedido.
- **Fórmula**: `valor_liquido_item = valor_bruto_item - ((valor_bruto_item / valor_bruto_total_pedido) * desconto_total_pedido)`.

### 2.2. SLA Logístico (Gold)
- **Regra**: Um pedido é considerado em atraso se a `data_entrega` for maior que a `data_previsao_entrega`.
- **Métrica**: Criamos a flag `flg_atraso` (1 ou 0) e o campo `dias_atraso` para facilitar o cálculo da taxa de serviço (OTD - On Time Delivery).

### 2.3. Padronização de Datas e Timestamps (Silver)
- **Problema**: Fontes com formatos heterogêneos (`dd/MM/yyyy`, ISO 8601 com 'T', etc).
- **Solução**: Criamos as funções `parse_multi_date` e `parse_multi_timestamp` que utilizam `try_to_date` e `try_to_timestamp` com múltiplos padrões, garantindo que o pipeline não quebre diante de novos formatos.

## 3. Qualidade de Dados (Data Quality)

### 3.1. Tratamento de Inconsistências
Durante a fase de exploração, identificamos e tratamos:
- **Campos Numéricos Malformados**: Strings como `'N/A'` ou números com decimais em campos inteiros (ex: `'3.0'`) foram limpos via Regex e funções de cast seguro (`clean_int`, `clean_numeric`).
- **Nulos Estratégicos**: Campos de texto nulos foram padronizados para `'NAO INFORMADO'` ou `'IGNORADO'` via função `standardize_null_text`.
- **Deduplicação**: Aplicamos *Window Functions* (`ROW_NUMBER`) para identificar e marcar duplicidades lógicas em itens de pedido.

### 3.2. Data Quality Gate (flg_qualidade)
Em vez de descartar dados, optamos pela estratégia de **Flags de Qualidade**. Registros que falham em validações críticas (ex: valores negativos, campos obrigatórios ausentes) recebem `flg_qualidade = False`. Isso permite que o analista de BI decida se quer incluir ou não esses dados em seus relatórios de auditoria.

## 4. Modelagem Analítica (Star Schema)

A camada Gold foi estruturada em um **Star Schema** para maximizar a performance de ferramentas como Power BI e Tableau.

- **Dimensões**: Atuam como filtros denormalizados. A `dim_vendedores` é o maior exemplo, consolidando 3 tabelas Silver em uma única estrutura de consulta.
- **Fatos**: Armazenam as métricas quantitativas. A separação entre `fact_vendas` e `fact_entregas` resolve o problema de granularidades distintas, evitando duplicidade em somatórios de frete e receita.

## 5. Validações Aplicadas
- Verificação de chaves primárias e estrangeiras.
- Validação de sanidade financeira (Valor Líquido <= Valor Bruto).
- Checagem de integridade de datas (Data de Entrega >= Data de Pedido).

## 6. Limitações e Evoluções Futuras
- **Limitação**: O ambiente Databricks Community não permite o uso de orquestradores nativos (Workflows).
- **Evolução**: Implementação de testes automatizados com **Great Expectations** ou **Soda Core** para validação contínua de contratos de dados (Data Contracts).
