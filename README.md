# Análise Northwind Traders

Relatório de indicadores de negócio para a Northwind Traders, uma loja fictícia de alimentos, bebidas e utilidades domésticas, construído a partir do clássico dataset Northwind — um exercício de modelagem analítica e geração de insights de negócio a partir de dados relacionais brutos.

📄 **[Relatório final (PDF)](./Relatorio_Northwind_Luciana_Simioni.pdf)**
📊 **[Planilha completa com dados e fórmulas](./northwind_analise.xlsx)**

---

## Contexto

A Northwind Traders é uma empresa fictícia com ~30 funcionários e faturamento mensal de ~R$1,5M, com clientes e fornecedores distribuídos em diversos países. O objetivo deste projeto foi simular um cenário comum em empresas em crescimento: dados dispersos em tabelas transacionais isoladas, sem uma camada analítica que permita responder perguntas de negócio de forma consolidada.

A partir de 14 tabelas brutas no formato de um ERP típico, o objetivo foi construir um relatório de indicadores capaz de apoiar dois objetivos estratégicos comuns em varejo: **aumentar o ticket médio** e **reduzir o churn de clientes**.

## O problema de dados

As tabelas fornecidas estavam na granularidade transacional típica de um sistema OLTP — cada uma representando uma entidade isolada (pedidos, itens de pedido, produtos, categorias, clientes), sem nenhuma camada analítica entre o banco e a análise. Isso significa que, antes de calcular qualquer indicador, foi necessário:

- Decidir a granularidade correta de análise (item de pedido, pedido, ou cliente)
- Relacionar 5 tabelas diferentes para obter cada métrica
- Tratar inconsistências de tipo, duplicidade e dados ausentes
- Criar campos derivados que nenhuma tabela tinha de forma direta (status de entrega, frequência de cliente, data quebrada em ano/mês/trimestre)

## Abordagem de modelagem

Trabalhei com Google Sheets, mas a lógica aplicada é a mesma de uma camada de transformação em SQL — só que implementada com fórmulas. A estrutura ficou assim:

```
orders ──┐
orders_details ──┤
product ─────────┼──►  pedidos  (tabela única, granularidade item-de-pedido)
categories ───────┤
customers ────────┘
```

**Tabelas-fonte** (`orders`, `orders_details`, `product`, `categories`, `customers`): mantidas como extraídas do ERP, sem alteração — equivalente a uma camada *staging*.

**Tabela `pedidos`**: uma única tabela analítica (*One Big Table*) na granularidade de item de pedido (2.155 linhas, uma por combinação `order_id` + `product_id`), construída cruzando as 5 tabelas-fonte. Essa foi a decisão de modelagem central do projeto — em vez de analisar cada tabela isoladamente, era necessário ter uma base só, na menor granularidade disponível, para poder agregar para qualquer nível depois (pedido, cliente, categoria, país, mês).

### Como os joins foram feitos

Cada coluna de `pedidos` que vem de outra tabela usa `VLOOKUP` para trazer o dado pela chave correta — o equivalente funcional de um `LEFT JOIN` em SQL:

| Coluna | Lógica | Equivalente em SQL |
|---|---|---|
| `customer_id` | `VLOOKUP(order_id, orders, ...)` | `JOIN orders ON order_id` |
| `product_name` | `VLOOKUP(product_id, product, ...)` | `JOIN product ON product_id` |
| `category_name` | `VLOOKUP(VLOOKUP(product_id → category_id), categories, ...)` | `JOIN product JOIN categories` (join encadeado) |
| `company_name` | `VLOOKUP(customer_id, customers, ...)` | `JOIN customers ON customer_id` |

O caso de `category_name` é o mais interessante: como a categoria não está na tabela de pedidos nem na de produtos de forma direta, foi necessário um lookup encadeado (`product_id → category_id → category_name`), simulando um join de duas tabelas em sequência.

### Campos calculados

Além dos joins, várias colunas foram derivadas para viabilizar os indicadores do relatório:

- **`total_item`** = `quantity × unit_price × (1 - discount)` — receita líquida por item, base de todo o cálculo de receita
- **`total_pedido`** = soma de `total_item` por `order_id` — receita agregada ao nível de pedido
- **`customer_frequency`** = contagem de pedidos por cliente — usado para identificar clientes recorrentes vs. esporádicos
- **`delivery_status`** = lógica condicional cruzando `shipped_date` x `required_date` → classifica em *On Time*, *Late* ou *Not Shipped*
- **`freight_unico`** / **`delivery_status_unique`** = colunas de deduplicação — como `pedidos` está na granularidade de item, métricas que pertencem ao pedido (frete, status de entrega) apareceriam duplicadas em cada item; essas colunas zeram o valor exceto na primeira ocorrência de cada `order_id`, evitando double counting nas agregações
- **`year`, `month`, `year_month`, `quarter`** = decomposição de `order_date` para permitir análise de série temporal

Esse cuidado com deduplicação foi importante: como a granularidade da base é item-de-pedido, qualquer métrica que existe no nível de pedido (frete, atraso) precisa ser contada uma única vez por pedido — senão indicadores como "frete total" ficam inflados artificialmente.

## Indicadores construídos

A partir da tabela `pedidos`, foram calculados:

- Evolução de pedidos, clientes ativos e frete por mês (1996–1998)
- Evolução de receita líquida mensal, com análise da queda atípica em maio/1998
- Taxa de entrega pontual (93%) e volume de pedidos atrasados/não enviados
- Ticket médio geral e por categoria de produto
- Volume de desconto concedido, total e por categoria
- Top 5 clientes por maior e menor ticket médio
- Distribuição de receita, clientes e frete médio por país (21 países)
- Análise de frete médio por país, sem relação clara com volume de pedidos

## Principais insights

- **Queda de receita em maio/1998** não foi um problema de ticket médio, mas de redução de 69% na base de clientes ativos no mês (ressalva: o mês de dados parece incompleto, com último pedido registrado em 06/05/1998).
- **Áustria** tem apenas 2 clientes mas gera a maior receita média por cliente de toda a base (~$64k/cliente) — sinal de que concentração em clientes de alto valor pode ser mais eficiente do que expansão de base em alguns mercados.
- **Meat/Poultry** tem o maior ticket médio entre categorias, mas também concentra os maiores volumes de desconto — oportunidade de revisar a política comercial da categoria.
- Problemas logísticos (37 pedidos atrasados, 21 não enviados) **não estão concentrados** em clientes de menor valor, o que descarta a hipótese de que atrasos seriam causa direta de queda de receita.

## Recomendações

- Reduzir pedidos em atraso/não enviados (quick win de experiência do cliente)
- Investigar gargalos operacionais na logística com dados mais granulares
- Trabalhar cross-sell e descontos progressivos para clientes de alta frequência e baixo ticket
- Avaliar competitividade de portfólio e preço nos mercados de maior receita (EUA, Alemanha)

## Ferramentas utilizadas

Google Sheets (`VLOOKUP`, `SUMIF`, `COUNTIF`, fórmulas de data e lógica condicional para construção da camada analítica e dos indicadores).

---

*Projeto pessoal de análise de dados.*
