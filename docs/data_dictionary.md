# Dicionário de Dados - Projeto Medallion

Este documento detalha as tabelas e campos presentes em cada camada do Data Lake (Bronze, Silver e Gold), servindo como guia de linhagem e semântica dos dados.

---

## 1. Camada Bronze (Raw Ingestion)
A camada Bronze armazena os dados em seu formato original, mas já convertidos para o formato Delta. O objetivo aqui é a **fidelidade total à fonte**.

- **Tabelas**: Correspondem aos nomes dos arquivos originais (ex: `vendedores`, `logistica_entregas`, `erp_pedidos_cabecalho_2025`).
- **Campos**: Mantêm os nomes e tipos originais da fonte (majoritariamente `string` para evitar perda de dados no cast).
- **Metadados**: Adicionamos `_bronze_processed_at` e `_source_file_name`.

---

## 2. Camada Silver (Refined / Data Quality)
Nesta camada, os dados são limpos, padronizados e tipados. Aqui aplicamos o **Data Quality Gate**.

### Tabela: `silver.clientes`
*Granularidade: 1 registro por cliente.*
- `customer_id`: ID único do CRM.
- `customer_name`: Nome normalizado (Upper Case).
- `segment`, `city`, `state`: Atributos geográficos e de segmentação.
- `registration_date`: Data de cadastro (Normalizada via `parse_multi_date`).
- `flg_ativo`: Booleano indicando status atual.
- `flg_qualidade`: `True` se o e-mail e ID estiverem preenchidos corretamente.

### Tabela: `silver.produtos`
*Granularidade: 1 registro por produto.*
- `product_id`: ID único do produto.
- `product_name`, `category`, `subcategory`, `brand`: Metadados do produto.
- `list_price`: Preço de lista (Normalizado via `clean_numeric`).
- `updated_at`: Última atualização (Normalizada via `parse_multi_timestamp`).
- `flg_qualidade`: `True` se o preço for positivo e a categoria estiver preenchida.

### Tabela: `silver.pedidos` (Cabeçalho)
*Granularidade: 1 registro por pedido.*
- `pedido_id`: Identificador único da transação.
- `customer_id`, `vendedor_id`: FKs para clientes e vendedores.
- `data_pedido`, `data_previsao_entrega`: Datas normalizadas.
- `valor_bruto`, `valor_desconto`, `valor_liquido`: Valores financeiros limpos.
- `flg_qualidade`: `True` se `valor_liquido <= valor_bruto` e datas forem lógicas.

### Tabela: `silver.pedidos_itens`
*Granularidade: 1 registro por item dentro do pedido.*
- `pedido_id`, `item_id`: Composta a PK do item.
- `product_id`: FK para produtos.
- `quantidade`, `preco_unitario`, `valor_item`: Métricas normalizadas via `clean_int` e `clean_numeric`.
- `flg_qualidade`: `True` se quantidade > 0 e não houver duplicidade lógica de item.

### Tabela: `silver.entregas`
*Granularidade: 1 registro por entrega.*
- `delivery_id`, `order_id`: Chaves de entrega e pedido.
- `shipped_at`, `delivered_at`: Timestamps normalizados (Tratamento do caractere 'T').
- `carrier_name`, `delivery_status`, `cost`: Dados logísticos.
- `flg_qualidade`: `True` se o custo for positivo e o ID do pedido existir.

---

## 3. Camada Gold (Curated / BI-Ready)
Camada final modelada em **Star Schema** para consumo analítico.

### Tabela: `gold.dim_vendedores`
*Consolidação de Vendedores + Canais + Regiões.*
- `seller_id`, `seller_name`: Dados do vendedor.
- `nome_canal`, `tipo_canal`: Informações do canal de venda.
- `regional_name`, `regional_state`, `manager_name`: Dados da gerência regional.

### Tabela: `gold.fact_vendas`
*Fato principal de vendas (Granularidade: Item).*
- `pedido_id`, `item_id`, `product_id`, `customer_id`, `vendedor_id`: Chaves para dimensões.
- `valor_bruto_item`: Valor total do item sem descontos.
- `valor_desconto_item`: Valor do desconto do cabeçalho rateado proporcionalmente.
- `valor_liquido_item`: Valor final real da venda do item.
- `flg_qualidade_venda`: Consolidação da qualidade do cabeçalho e do item.

### Tabela: `gold.fact_entregas`
*Fato focada em SLA logístico.*
- `pedido_id`, `delivery_id`: Chaves de ligação.
- `data_previsao_entrega`, `data_entrega`: Datas para cálculo de SLA.
- `dias_atraso`: Diferença calculada de atraso.
- `flg_atraso`: Flag binária para facilidade de agregados em BI.
- `custo_frete`: Custo da operação logística.

---

## 4. Campos Técnicos de Linhagem
Todas as tabelas Silver e Gold possuem os campos:
- `_silver_processed_at` / `_gold_processed_at`: Data/Hora exata do processamento da carga.
- `flg_qualidade`: Indicador de confiança do registro.
