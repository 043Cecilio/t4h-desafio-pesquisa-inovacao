# Diagnóstico e Otimização de Performance em API Node.js e PostgreSQL sob Restrição de Recursos

## 1. Resumo Executivo

Este artigo apresenta o estudo de caso prático de diagnóstico e otimização de performance de um sistema backend de e-commerce composto por uma API REST em **Node.js 20 (Express)** e um banco de dados relacional **PostgreSQL 16**, executados em containers Docker com limites estritos de hardware (**1,0 CPU / 512 MiB RAM** para o banco de dados e **0,5 CPU / 256 MiB RAM** para a API) sobre uma massa de **2,12 milhões de registros**.

Seguindo um método experimental rigoroso — baseado na preservação estrita do contrato HTTP/JSON (42 casos verificados automaticamente), isolamento de variáveis (uma mudança por commit) e avaliação pela **mediana de 3 execuções** de carga no k6 (20 usuários virtuais durante 60 segundos) — evoluímos a aplicação do estado inicial (**4,41 req/s**, latência mediana de **691,1 ms** e p95 de **15.274,7 ms**) para **18,55 a 19,37 req/s** (**+339,2% de vazão**), com latência mediana geral de **92,5 ms** (**-86,6%**), queda de **99,2%** na mediana da rota crítica de pedidos por cliente (de **11.913,6 ms para 92,9 ms**) e redução de **83,5%** no consumo de CPU e **72,0%** no pico de memória RAM do container da API.

---

## 2. Ambiente, Carga de Trabalho e Metodologia

A base de dados inicializa deterministicamente quatro tabelas relacionais (`banco/01-estrutura.sql` e `banco/02-dados.sql`):
* **`clientes`**: 100.000 registros;
* **`produtos`**: 20.000 registros;
* **`pedidos`**: 500.000 registros (distribuídos entre os status `pago`, `entregue`, `pendente` e `cancelado`, contendo uma coluna textual longa `observacoes`);
* **`itens_pedido`**: 1.500.000 registros (média de 3 itens por pedido).

A carga sintética (`/carga/teste-carga.js`) simula 20 usuários virtuais concorrentes (`VUS=20`) com semente pseudoaleatória fixa (`SEMENTE=42`), distribuindo as chamadas entre 7 rotas:
* `GET /clientes/:id` (`cliente` — 25% do tráfego)
* `GET /clientes/:id/pedidos` (`pedidos-do-cliente` — 20% do tráfego)
* `GET /clientes?email=` (`cliente-por-email` — 15% do tráfego)
* `GET /pedidos?status=&pagina=` (`pedidos-por-status` — 15% do tráfego)
* `GET /produtos/busca?q=` (`busca-produtos` — 15% do tráfego)
* `GET /relatorios/vendas-por-dia` (`vendas-por-dia` — 8% do tráfego)
* `GET /relatorios/top-produtos` (`top-produtos` — 2% do tráfego)

Para garantir validade científica, todas as medições respeitaram três critérios:
1. **Contrato Imutável:** Antes da primeira alteração, as 42 respostas de referência foram gravadas em `contrato/base.json`. Todas as baterias posteriores validaram 100% de igualdade em status HTTP e corpo JSON (`contrato.txt`).
2. **Rastreabilidade de Código:** Cada experimento foi commitado antes da execução do `scripts/medir.sh`, assegurando `arquivos alterados sem commit: 0` nos arquivos `contexto.txt`.
3. **Mediana de 3 Baterias:** Para eliminar variações de escalonamento de containers, cada hipótese foi medida 3 vezes consecutivas (com 15 segundos de aquecimento prévio em cada rodada), adotando-se a mediana como métrica de decisão.

---

## 3. Diagnóstico da Linha de Base (`00-linha-de-base`)

Nas três medições da linha de base (`resultados/20261107-173622-00-linha-de-base`, `173758` e `173932`), o sistema apresentou severa degradação:
* **Vazão Mediana:** `4,41 req/s` (290 requisições atendidas em 60s).
* **Latência Geral:** Mediana de `691,1 ms`, p95 de `15.274,7 ms` e p99 de `19.311,1 ms`.
* **Consumo de Recursos (`recursos.txt`):** O container `banco` operou estrangulado em **98,2% de CPU média** (pico de **99,8%**, esgotando 1,0 CPU inteira) e **426,1 MiB de RAM** (pico de 472,4 MiB em 512 MiB disponíveis), enquanto o container `api` utilizou apenas **8,3% de CPU média** e **34,6 MiB de RAM**, permanecendo ocioso à espera de respostas do banco de dados.

A inspeção do código-fonte (`api/src/`) combinada com a coleta das estatísticas internas do PostgreSQL (`pg_stat_user_tables`) e dos planos de execução reais (`EXPLAIN (ANALYZE, BUFFERS)`), salvos em `diagnostico/01-estatisticas-e-indices-iniciais.txt` e `diagnostico/02-explain-analyze-linha-de-base.txt`, comprovou quatro causas-raiz:

### 3.1. Ausência Total de Índices Secundários e Varreduras Bilionárias
A consulta ao catálogo `pg_indexes` revelou que o banco possuía exclusivamente os 4 índices de chave primária (`clientes_pkey`, `pedidos_pkey`, `itens_pedido_pkey` e `produtos_pkey`). Nenhuma chave estrangeira ou coluna de filtro estava indexada. Como consequência, após a linha de base, o catálogo `pg_stat_user_tables` registrava:

| Tabela | Linhas (`n_live_tup`) | `seq_scan` | Tuplas Lidas em Seq Scan (`seq_tup_read`) | `idx_scan` |
| :--- | :---: | :---: | :---: | :---: |
| `itens_pedido` | 1.500.000 | **2.579** | **1.960.500.000** | **0** |
| `pedidos` | 500.000 | **1.287** | **325.000.000** | 1.500.264 |
| `clientes` | 100.000 | **170** | **16.900.000** | 500.497 |
| `produtos` | 20.000 | **283** | **5.640.000** | 1.503.549 |

Quase **2 bilhões de linhas** foram varridas sequencialmente em `itens_pedido` em poucos minutos de teste porque cada busca de itens de um pedido (`WHERE pedido_id = $1`) executava um `Parallel Seq Scan` sobre 1,5 milhão de linhas (**115,599 ms** por execução isolada no `EXPLAIN ANALYZE`, lendo 9.555 blocos de 8 KB).

### 3.2. Instanciação de Conexão TCP/Backend por Consulta (`api/src/db.js`)
Em `api/src/db.js`, a função `consultar()` instanciava `new Client()`, invocava `await cliente.connect()` e `await cliente.end()` no bloco `finally`. Em vez de reutilizar conexões persistentes, cada query individual sofria a latência de abertura de socket TCP, autenticação e criação de um processo *backend* no PostgreSQL.

### 3.3. Consultas N+1 na Rota de Maior Peso (`GET /clientes/:id/pedidos`)
Na rota `pedidos-do-cliente` (mediana de **11.913,6 ms** e p95 de **19.126,2 ms** na base), o código executava:
1. `SELECT * FROM clientes WHERE id = $1` (1 query);
2. `SELECT * FROM pedidos WHERE cliente_id = $1` (1 query — **336,061 ms** em `Parallel Seq Scan` sobre 500.000 linhas);
3. Para cada pedido retornado ($N$), uma query `SELECT * FROM itens_pedido WHERE pedido_id = $1` (**115,599 ms** cada em `Parallel Seq Scan`);
4. Para cada item ($M$), uma query `SELECT nome FROM produtos WHERE id = $1`.

Um cliente com 5 pedidos e 15 itens disparava **22 consultas SQL sequenciais** na mesma requisição HTTP, cada uma abrindo e fechando uma conexão nova no banco.

### 3.4. Predicado Não-SARGable e Agregação em Memória na API (`GET /relatorios/vendas-por-dia`)
A rota `/relatorios/vendas-por-dia` apresentava dois defeitos arquiteturais:
1. Filtrava o período com `WHERE p.criado_em::date BETWEEN $1 AND $2`. Aplicar o *cast* `::date` sobre a coluna `p.criado_em` impede que o otimizador do PostgreSQL utilize um índice B-Tree padrão na coluna `timestamp`, forçando um `Parallel Seq Scan`.
2. A consulta buscava todas as linhas desagregadas do período (`12.945 linhas` para um intervalo de 7 dias no `EXPLAIN ANALYZE`, levando **283,977 ms**), projetando inclusive a coluna larga `p.observacoes` (`width=297`), e transferia esse volume pela rede para o Node.js instanciar objetos `Map` e `Set` e somar valores no *Event Loop*.

---

## 4. Intervenções Realizadas e Resultados Experimentais

### 4.1. Experimento 01: Pool de Conexões (`01-pool-conexoes`)
* **Implementação:** Substituição de `pg.Client` por `pg.Pool` (`max: 20`) em `api/src/db.js`.
* **Resultado:** A vazão subiu de **4,41 para 5,50 req/s (+24,7%)** e a mediana geral caiu de **691,1 ms para 451,4 ms (-34,7%)**. O impacto mais visível ocorreu nas rotas cujas queries já utilizavam chave primária ou tabelas menores: a mediana de `GET /clientes/:id` caiu de **292,4 ms para 98,4 ms (-66,3%)**, e `GET /produtos/busca` caiu de **394,3 ms para 188,2 ms (-52,3%)**.

### 4.2. Experimento 02: Indexação Relacional e Funcional (`02-indices`)
* **Implementação:** Criação do script `banco/03-indices.sql` contendo seis índices B-Tree:
  * `idx_clientes_lower_email ON clientes (lower(email))` — índice funcional casando exatamente com a expressão `WHERE lower(email) = lower($1)`;
  * `idx_pedidos_cliente_id ON pedidos (cliente_id)`;
  * `idx_pedidos_status ON pedidos (status)`;
  * `idx_pedidos_criado_em ON pedidos (criado_em)`;
  * `idx_itens_pedido_pedido_id ON itens_pedido (pedido_id)`;
  * `idx_itens_pedido_produto_id ON itens_pedido (produto_id)`.
* **Resultado:** O efeito foi transformador. A vazão saltou de **5,50 para 18,54 req/s (+237,1% em relação ao Exp 01 e +320,4% sobre a linha de base)**, atendendo **1.320 requisições** em 60 segundos contra 290 da linha de base. A latência mediana geral despencou para **101,4 ms (-85,3% sobre a base)** e o p95 geral caiu pela metade (**de 15.274,7 ms para 6.416,5 ms**). Na rota `pedidos-do-cliente`, a substituição dos *Parallel Seq Scans* por *Index Scans* reduziu a mediana de **9.183,7 ms para 291,8 ms (-96,8%)**.

### 4.3. Experimento 03: Resolução do N+1 com `LEFT JOIN` (`03-join-pedidos-cliente`)
* **Implementação:** Reescrita da rota `GET /clientes/:id/pedidos` em `api/src/rotas.js