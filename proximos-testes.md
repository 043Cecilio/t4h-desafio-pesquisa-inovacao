# Próximos Testes e Ciclo de Otimização Futuro

Após os 4 experimentos implementados, 4 das 7 rotas da API respondem entre **3,8 ms e 96,7 ms** na mediana (`cliente`, `cliente-por-email`, `pedidos-do-cliente` e `busca-produtos`). No entanto, o container do PostgreSQL permanece operando com **~100% de uso de 1 CPU** (`recursos.txt`), concentrado nas três operações analíticas e de paginação profunda: `top-produtos` (mediana de 16,8 s), `pedidos-por-status` (p95 de 12,5 s) e o join residual de `vendas-por-dia` (mediana de 2,49 s).

Abaixo estão os próximos experimentos priorizados por impacto esperado, com hipótese, implementação e forma de medição.

---

## Prioridade 1 — Pré-agregação / Visão Materializada para `GET /relatorios/top-produtos`

* **Gargalo Alvo:** A rota `/relatorios/top-produtos` representa apenas 2% das requisições do teste de carga (~21 chamadas por minuto), mas cada execução realiza um `Parallel Hash Join` entre `pedidos` (filtrando `status <> 'cancelado'`, o que abrange **90% da tabela / 450.000 pedidos**), `itens_pedido` (**1.350.000 itens**) e `produtos` (**20.000 produtos**), seguido de `HashAggregate`. No `EXPLAIN ANALYZE` isolado (`diagnostico/02-explain-analyze-linha-de-base.txt`), uma única chamada consome **1.011,9 ms** e lê mais de 30.000 blocos de buffer. Sob concorrência, seu p95 chega a **32,3 segundos**.
* **Hipótese:** Como a condição `status <> 'cancelado'` seleciona 90% da base, nenhum índice B-Tree convencional evita a varredura de 1,35 milhão de itens. Criar uma visão materializada (`CREATE MATERIALIZED VIEW mv_top_produtos`) — ou tabela de resumo mantida por *trigger* em cargas de escrita — pré-agregando `unidades` e `receita_centavos` por produto reduzirá o tempo de execução de `top-produtos` de **> 1.000 ms para < 2 ms** (leitura indexada de no máximo 50 linhas), liberando a maior fatia de CPU do PostgreSQL.
* **Implementação Proposta (`banco/04-mv-top-produtos.sql`):**
  ```sql
  CREATE MATERIALIZED VIEW mv_top_produtos AS
  SELECT pr.id AS produto_id,
         pr.nome,
         pr.categoria,
         sum(i.quantidade)::bigint AS unidades,
         sum(i.quantidade * i.preco_unitario_centavos)::bigint AS receita_centavos
    FROM itens_pedido i
    JOIN pedidos p ON p.id = i.pedido_id
    JOIN produtos pr ON pr.id = i.produto_id
   WHERE p.status <> 'cancelado'
   GROUP BY pr.id, pr.nome, pr.categoria;

  CREATE UNIQUE INDEX idx_mv_top_produtos_receita
    ON mv_top_produtos (receita_centavos DESC, produto_id ASC);
  ```
* **Como testar:** Apontar a consulta de `GET /relatorios/top-produtos` para `mv_top_produtos`, validar as 42 respostas com `scripts/verificar-contrato.sh comparar` e rodar 3 medições com `scripts/medir.sh 05-mv-top-produtos`.

---

## Prioridade 2 — Índice Composto de Cobertura (*Index-Only Scan*) e `VACUUM ANALYZE` em `GET /pedidos`

* **Gargalo Alvo:** A rota `GET /pedidos?status=...&pagina=...` (`pedidos-por-status`, 15% da carga) possui mediana de **595,3 ms**, mas seu p95 atinge **12.578,5 ms**. Isso ocorre por três motivos evidenciados no `EXPLAIN ANALYZE`:
  1. A rota executa duas queries por requisição: `SELECT count(*) FROM pedidos WHERE status = $1` (varrendo ~100.000 a 200.000 registros por status) e `SELECT * FROM pedidos WHERE status = $1 ORDER BY criado_em DESC, id DESC LIMIT 20 OFFSET $3`.
  2. O índice atual `idx_pedidos_status` indexa apenas `(status)`. O PostgreSQL precisa buscar dezenas de milhares de tuplas no *heap* e realizar um *Sort* em memória por `criado_em DESC, id DESC`.
  3. O uso de `SELECT *` obriga a leitura da coluna de texto `observacoes` (`width=299`), embora a resposta JSON retorne apenas `id, cliente_id, status, criado_em`. Além disso, sem rodar `VACUUM ANALYZE` após o `02-dados.sql`, o *Visibility Map* não está preenchido, impedindo *Index-Only Scans*.
* **Hipótese:** Substituir `SELECT *` por `SELECT id, cliente_id, status, criado_em`, criar o índice composto de cobertura `CREATE INDEX idx_pedidos_status_criado_id ON pedidos (status, criado_em DESC, id DESC) INCLUDE (cliente_id)` e executar `VACUUM ANALYZE` na inicialização permitirá que tanto o `count(*)` quanto a paginação ordenada sejam resolvidos inteiramente via **Index-Only Scan** sem etapa de *Sort*.
* **Como testar:** Adicionar o índice composto e `VACUUM ANALYZE` em `banco/04-indices-cobertura.sql`, ajustar as colunas projetadas em `api/src/rotas.js`, validar o contrato e medir 3 vezes (`scripts/medir.sh 06-indice-cobertura-pedidos`).

---

## Prioridade 3 — Índices de Cobertura para o Join de `GET /relatorios/vendas-por-dia`

* **Gargalo Alvo:** No Experimento 04, `vendas-por-dia` caiu de 5,6 s para 2,49 s na mediana, mas ainda realiza leituras no *heap* de `itens_pedido` para buscar `quantidade` e `preco_unitario_centavos` dos pedidos do período.
* **Hipótese:** Criar um índice parcial de cobertura em `pedidos` (`CREATE INDEX idx_pedidos_ativos_criado_em ON pedidos (criado_em, id) WHERE status <> 'cancelado'`) em conjunto com um índice de cobertura em `itens_pedido` (`CREATE INDEX idx_itens_pedido_cobertura ON itens_pedido (pedido_id) INCLUDE (quantidade, preco_unitario_centavos)`) permitirá que o join e a soma financeira ocorram 100% via *Index-Only Scan*, reduzindo a latência de `vendas-por-dia` de ~2,49 s para menos de 100 ms.

---

## Prioridade 4 — Índice GIN de Trigramas (`pg_trgm`) em `GET /produtos/busca` e Remoção de Serialização no Logger

* **Gargalo Alvo:**
  1. A busca textual `WHERE nome ILIKE '%' || $1 || '%'` impede o uso de índices B-Tree comuns, exigindo um `Seq Scan` nas 20.000 linhas de `produtos` (~10,9 ms isolado, ~96,7 ms sob carga).
  2. Em `api/src/server.js`, o middleware intercepta `res.json` para serializar o payload completo (`JSON.stringify(corpo).length`) e emitir `console.log` síncrono em todas as requisições.
* **Hipótese:** Habilitar a extensão `pg_trgm` com índice `USING gin (nome gin_trgm_ops)` eliminará o `Seq Scan` em `produtos`, e substituir a serialização duplicada em `server.js` pela leitura do cabeçalho `res.get('content-length')` reduzirá ainda mais o custo de CPU e *Garbage Collection* no Node.js.