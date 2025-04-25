# # Desafio Técnico SQL – Systock (Consumo, Requisição e Validação de Dados)

Este repositório contém soluções SQL para o desafio técnico proposto pela Systock. O foco principal está em análises de consumo, requisições pendentes, movimentação de produtos e validação de dados para o mês de fevereiro de 2025, com foco em consistência e integridade das informações.

Tecnologias e Ferramentas
- PostgreSQL
- SQL puro
- VS Code / DBeaver (ou a IDE que você usou)
 1.1 – Consumo por produto e mês

Organização
- Parte 1 – Consultas SQL
- Parte 2 – Transformações de Dados
- Parte 3 – Estratégia de Validação com o Cliente
 Autor
- Moisés [LinkedIn](https://www.linkedin.com/in/mois%C3%A9spcastro31/)





1-Consulta que soma o total vendido por produto no mês de fevereiro de 2025.

SELECT 
    produto_id,
    SUM(qtde_vendida) AS total_consumo
FROM 
    venda
WHERE 
    data_emissao >= '2025-02-01' AND data_emissao < '2025-03-01'
GROUP BY 
    produto_id
ORDER BY 
    total_consumo DESC;

1.2 – Produtos com requisição pendente
Lista os produtos que foram solicitados em pedidos de compra e ainda não foram recebidos (ausentes na tabela de entradas).

SELECT 
    pc.ordem_compra,
    pc.produto_id,
    pc.qtde_pedida,
    COALESCE(SUM(em.qtde_recebida), 0) AS total_recebido
FROM 
    pedido_compra pc
LEFT JOIN entradas_mercadoria em 
    ON pc.ordem_compra::varchar = em.ordem_compra AND pc.produto_id = em.produto_id
GROUP BY 
    pc.ordem_compra, pc.produto_id, pc.qtde_pedida
HAVING 
    COALESCE(SUM(em.qtde_recebida), 0) < pc.qtde_pedida;

 1.3 – Produtos requisitados mas não consumidos
Filtra produtos que foram solicitados em fevereiro, mas que nem foram recebidos nem vendidos no mesmo período.

SELECT DISTINCT
    pc.ordem_compra,
    pc.produto_id,
    pc.qtde_pedida
FROM 
    pedido_compra pc
LEFT JOIN entradas_mercadoria em 
    ON pc.ordem_compra::varchar = em.ordem_compra 
    AND pc.produto_id = em.produto_id
    AND em.data_entrada >= '2025-02-01' AND em.data_entrada < '2025-03-01'
LEFT JOIN venda v 
    ON pc.produto_id = v.produto_id 
    AND v.data_emissao >= '2025-02-01' AND v.data_emissao < '2025-03-01'
WHERE 
    em.entrada_id IS NULL
    AND v.venda_id IS NULL;

 Parte 2 – Transformações de Dados
Transformação de dados
Agrega os produtos com mais de 10 unidades requisitadas, agrupando por data e nome formatado do produto.

SELECT 
    pc.produto_id || ' - ' || pc.descricao_produto AS produto,
    SUM(pc.qtde_pedida) AS qtde_requisitada,
    TO_CHAR(pc.data_pedido, 'DD/MM/YYYY') AS data_solicitacao
FROM pedido_compra pc
WHERE pc.data_pedido >= '2025-01-01' AND pc.data_pedido < '2025-03-01'
GROUP BY 
    pc.produto_id, pc.descricao_produto, pc.data_pedido
HAVING SUM(pc.qtde_pedida) > 10
ORDER BY qtde_requisitada DESC;


## Parte 3 – Estratégia de Validação com o Cliente

Imagine que você precisa validar os dados do mês de **Fevereiro de 2025** com o cliente.

### Tarefa
1-Quais seriam os principais pontos que você validaria com o cliente?
 a. Consistência de períodos
Confirmar se o período de análise está correto: 01/02/2025 a 28/02/2025

Verificar se há lançamentos fora do período que foram erroneamente incluídos ou omitidos.

 b. Volumes de consumo
Validar os produtos mais consumidos e suas quantidades.

Comparar com o histórico ou a expectativa do cliente.

 c. Requisições pendentes
Garantir que os produtos com qtde_pendente > 0 não foram recebidos nem consumidos.

 d. Produtos sem movimentação
Validar produtos requisitados no mês que não foram entregues nem consumidos.

 e. Possíveis erros de entrada
Conferir produtos com valores inconsistentes (ex: quantidades negativas, ou requisições acima da média).

2-Quais técnicas utilizaria para garantir a exatidão e a precisão dos dados?
✅ Validações cruzadas
Comparar os dados entre as tabelas pedido_compra, entradas_mercadoria e venda.

✅ Checagens de integridade
Garantir que produto_id exista em todas as tabelas relevantes.

Conferir se existem NULLs onde não deveriam existir (ex: quantidades, datas, IDs).

✅ Limpeza de dados
Detectar e remover registros duplicados.

Tratar campos com tipos incorretos (ex: datas salvas como texto).

✅ Conferência com usuário-chave
Mostrar ao cliente os totais e permitir que ele compare com controles manuais ou planilhas internas.

3-Quais consultas você deixaria prontas para usar na reunião de validação?

a-Resumo de consumo por produto

SELECT
    produto_id,
    SUM(qtde_vendida) AS total_consumo
FROM venda
WHERE data_emissao BETWEEN '2025-02-01' AND '2025-02-28'
GROUP BY produto_id

b-Requisições pendentes ainda não recebidas

SELECT
    pc.produto_id,
    pc.descricao_produto,
    pc.ordem_compra,
    pc.qtde_pedida,
    pc.qtde_pendente
FROM pedido_compra pc
LEFT JOIN entradas_mercadoria em
    ON pc.ordem_compra::varchar = em.ordem_compra
    AND pc.produto_id = em.produto_id
WHERE pc.qtde_pendente > 0
  AND em.produto_id IS NULL
ORDER BY pc.qtde_pendente DESC;

c-Produtos requisitados mas não consumidos nem recebidos

SELECT DISTINCT
    pc.produto_id,
    pc.descricao_produto,
    pc.ordem_compra,
    pc.qtde_pedida,
    pc.qtde_pendente
FROM pedido_compra pc
LEFT JOIN entradas_mercadoria em
    ON pc.ordem_compra::varchar = em.ordem_compra
    AND pc.produto_id = em.produto_id
LEFT JOIN venda v
    ON pc.produto_id = v.produto_id
    AND v.data_emissao BETWEEN '2025-02-01' AND '2025-02-28'
WHERE pc.data_pedido BETWEEN '2025-02-01' AND '2025-02-28'
  AND em.produto_id IS NULL
  AND v.produto_id IS NULL;

d-Totais por dia para detecção de picos ou anomalias

SELECT
    data_emissao,
    COUNT(*) AS total_vendas,
    SUM(qtde_vendida) AS qtde_total
FROM venda
WHERE data_emissao BETWEEN '2025-02-01' AND '2025-02-28'
GROUP BY data_emissao
ORDER BY data_emissao;

e-Produtos com mais de 10 unidades requisitadas

SELECT
    pc.produto_id || ' - ' || pc.descricao_produto AS produto,
    SUM(pc.qtde_pedida) AS qtde_requisitada,
    TO_CHAR(pc.data_pedido, 'DD/MM/YYYY') AS data_solicitacao
FROM pedido_compra pc
WHERE pc.data_pedido BETWEEN '2025-01-01' AND '2025-03-31'
GROUP BY pc.produto_id, pc.descricao_produto, pc.data_pedido
HAVING SUM(pc.qtde_pedida) > 10
ORDER BY qtde_requisitada DESC;
