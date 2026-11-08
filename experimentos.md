# Registro de Experimentos de Otimização

Este documento detalha todos os ciclos experimentais realizados na aplicação, seguindo a metodologia de **uma mudança isolada por vez**, commit prévio à medição (`arquivos alterados sem commit: 0` em `contexto.txt`), verificação das 42 asserções de contrato (`scripts/verificar-contrato.sh comparar`) e avaliação pela **mediana de 3 execuções** de 60 segundos com 20 usuários virtuais (`VUS=20`, `AQUECIMENTO=15s`, `SEMENTE=42`).

---

## Resumo Consolidado (Mediana de 3 Medições por Rótulo)

| Rótulo do Experimento | Commit Base | Req Totais | Vazão (req/s) | Mediana Geral | p95 Geral | p99 Geral | CPU Média Banco | Mem Máx Banco | CPU Média API | Mem Máx API | Contrato |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`00-linha-de-base`** | `3538269` | 290 | 4,41 | 691,1 ms | 15.274,7 ms | 19.311,1 ms | 98,2% | 426,1 MiB | 8,3% | 45,2 MiB | Mantido (42/42) |
| **`01-pool-conexoes`** | `01-pool` | 350 | 5,50 | 451,4 ms | 12.662,6 ms | 16.915,0 ms | 98,4% | 414,5 MiB | 6,3% | 48,0 MiB | Mantido (42/42) |
| **`02-indices`** | `02-idx` | 1.320 | 18,54 | 101,4 ms | 6.416,5 ms | 13.344,9 ms | 99,8% | 460,5 MiB | 20,4% | 90,7 MiB | Mantido (42/42) |
| **`03-join-pedidos-cliente`** | `03-join` | 1.317 | **19,37** | 95,4 ms | **6.233,7 ms** | **13.101,3 ms** | 99,5% | 434,7 MiB | 20,0% | 92,5 MiB | Mantido (42/42) |
| **`04-sql-vendas-por-dia`** | `83f24f4` | 1.244 | 18,55 | **92,5 ms** | 6.965,5 ms | 16.868,8 ms | 100,3% | 466,7 MiB | **3,3%** | **25,9 MiB** | Mantido (42/42) |

---

## Linha de Base (`00-linha-de-base`)

* **Pastas de evidência:**
  * `resultados/20261107-173622-00-linha-de-base/` (`4,05 req/s` | `mediana 884,9 ms` | `p95 16.333,5 ms`)
  * `resultados/20261107-173758-00-linha-de-base/` (`4,53 req/s` | `mediana 634,5 ms` | `p95 15.274,7 ms`)
  * `resultados/20261107-173932-00-linha-de-base/` (`4,41 req/s` | `mediana 691,1 ms` | `p95 14.901,2 ms`)
* **Diagnóstico coletado (`diagnostico/01-estatisticas-e-indices-iniciais.txt` e `02-explain-analyze-linha-de-base.txt`):**
  * O container `banco` operou saturado em **98,2% de CPU média** (pico de **99,8%**) devido à ausência total de índices secundários e em chaves estrangeiras.
  * Em apenas 3 baterias de teste, `itens_pedido` acumulou **2.579 *Sequential Scans*** e **1.960.500.000 de tuplas lidas sequencialmente** (`idx_scan = 0`), enquanto `pedidos` acumulou **1.287 *Sequential Scans*** (**325.000.000 de tuplas lidas**).

---

## Experimento 01 — Pool de Conexões no PostgreSQL (`01-pool-conexoes`)

* **Hipótese:** A função `consultar()` em `api/src/db.js` instancia um novo `pg.Client()`, abre conexão TCP, autentica e encerra a sessão no PostgreSQL a cada query executada. Substituir `pg.Client` por um pool reutilizável (`pg.Pool` com `max: 20`) deve reduzir a latência das rotas rápidas e elevar a vazão geral ao eliminar o custo de criação de processos *backend* no PostgreSQL por consulta.
* **Mudança realizada:** Substituição de `Client` por `Pool` em `api/src/db.js`.
* **Pasta de evidência:** `resultados/*01-pool-conexoes/`.
* **Medição (Antes → Depois):**
  * **Vazão geral:** `4,41 req/s` → **`5,50 req/s`** (**+24,7%**)
  * **Latência mediana geral:** `691,1 ms` → **`451,4 ms`** (**-34,7%**)
  * **Latência p95 geral:** `15.274,7 ms` → **`12.662,6 ms`** (**-17,1%**)
  * **Rota `cliente` (`GET /clientes/:id`):** mediana caiu de `292,4 ms` para **`98,4 ms`** (**-66,3%**) e p95 de `601,6 ms` para **`222,4 ms`**.
  * **Rota `busca-produtos`:** mediana caiu de `394,3 ms` para **`188,2 ms`** (**-52,3%**).
* **Decisão:** **Mantida.** O ganho nas rotas leves confirmou a hipótese, enquanto a CPU do banco permaneceu em `98,4%` devido aos *Sequential Scans* ainda presentes nas tabelas.

---

## Experimento 02 — Criação de Índices B-Tree e Funcionais (`02-indices`)

* **Hipótese:** Criar índices B-Tree nas chaves estrangeiras (`itens_pedido(pedido_id)`, `itens_pedido(produto_id)`, `pedidos(cliente_id)`), nas colunas de filtro (`pedidos(status)`, `pedidos(criado_em)`) e um índice funcional em `clientes(lower(email))` deve substituir os *Sequential Scans* por *Index Scans*, reduzindo drasticamente a latência das rotas de consulta de clientes e pedidos.
* **Mudança realizada:** Criação do arquivo `banco/03-indices.sql` e recriação limpa do volume (`docker compose down -v && docker compose up -d --build`).
* **Pasta de evidência:** `resultados/*02-indices/`.
* **Medição (Antes [`01`] → Depois [`02`]):**
  * **Vazão geral:** `5,50 req/s` → **`18,54 req/s`** (**+237,1% sobre o Exp 01 / +320,4% sobre a linha de base**)
  * **Latência mediana geral:** `451,4 ms` → **`101,4 ms`** (**-77,5%**)
  * **Latência p95 geral:** `12.662,6 ms` → **`6.416,5 ms`** (**-49,3%**)
  * **Rota `pedidos-do-cliente`:** mediana despencou de `9.183,7 ms` para **`291,8 ms`** (**-96,8%**) e p95 de `16.784,2 ms` para **`709,9 ms`**.
  * **Rota `cliente-por-email`:** mediana caiu de `398,2 ms` para **`72,0 ms`** (**-81,9%**) e p95 de `1.090,8 ms` para **`177,1 ms`**.
  * **Rota `cliente`:** mediana caiu de `98,4 ms` para **`8,9 ms`** (**-91,0%**).
* **Decisão:** **Mantida.** Foi a mudança de maior impacto estrutural no throughput do sistema.

---

## Experimento 03 — Eliminação de N+1 Queries em Pedidos do Cliente (`03-join-pedidos-cliente`)

* **Hipótese:** Mesmo com índices, a rota `GET /clientes/:id/pedidos` executa $1 + 1 + N + M$ consultas separadas em laços `for` (buscando pedidos, depois itens de cada pedido e depois o produto de cada item). Consolidar a busca de pedidos, itens e nomes de produtos em uma única consulta SQL com `LEFT JOIN` e agregação em `Map` na API deve reduzir as viagens de rede (*round-trips*) e diminuir a latência da rota em mais de 50%.
* **Mudança realizada:** Refatoração do handler `GET /clientes/:id/pedidos` em `api/src/rotas.js` usando `LEFT JOIN itens_pedido` e `LEFT JOIN produtos`, omitindo a leitura da coluna não utilizada `p.observacoes`.
* **Pastas de evidência:**
  * `resultados/20261107-212159-03-join-pedidos-cliente/` (`19,37 req/s` | `p95 rota: 314,0 ms`)
  * `resultados/20261107-212332-03-join-pedidos-cliente/` (`20,15 req/s` | `p95 rota: 290,2 ms`)
  * `resultados/20261107-212501-03-join-pedidos-cliente/` (`18,90 req/s` | `p95 rota: 288,8 ms`)
* **Medição (Antes [`02`] → Depois [`03` — Mediana]):**
  * **Rota `pedidos-do-cliente` (Mediana):** `291,8 ms` → **`95,1 ms`** (**-67,4%**)
  * **Rota `pedidos-do-cliente` (p95):** `709,9 ms` → **`290,2 ms`** (**-59,1%**)
  * **Vazão geral:** `18,54 req/s` → **`19,37 req/s`** (pico de `20,15 req/s`)
  * **Latência p95 geral:** `6.416,5 ms` → **`6.233,7 ms`**
* **Decisão:** **Mantida.** A rota `pedidos-do-cliente` (que representa 20% do tráfego total do k6) passou a responder em menos de 100 ms na mediana.

---

## Experimento 04 — Predicado SARGable e Agregação SQL em Vendas por Dia (`04-sql-vendas-por-dia`)

* **Hipótese:** A rota `GET /relatorios/vendas-por-dia` apresenta dois problemas: (1) filtra a data usando `WHERE p.criado_em::date BETWEEN $1 AND $2`, um predicado não-SARGable que impede o uso do índice `idx_pedidos_criado_em`; e (2) transfere ~12.945 linhas brutas (incluindo `p.observacoes`) para a memória da API agrupar em JavaScript. Substituir o filtro por intervalo direto de timestamp (`p.criado_em >= $1::timestamp AND p.criado_em < ($2::date + INTERVAL '1 day')`) e realizar o `GROUP BY` com `COUNT(DISTINCT p.id)` e `SUM()` diretamente no PostgreSQL deve reduzir a latência da rota e derrubar o uso de CPU e memória do Node.js.
* **Mudança realizada:** Refatoração da consulta SQL e remoção do loop de agregação em memória na rota `GET /relatorios/vendas-por-dia` em `api/src/rotas.js`.
* **Pastas de evidência:**
  * `resultados/20261107-213918-04-sql-vendas-por-dia/` (`19,21 req/s` | `vendas-por-dia mediana: 2.615,7 ms`)
  * `resultados/20261107-214052-04-sql-vendas-por-dia/` (`17,23 req/s` | `vendas-por-dia mediana: 2.492,9 ms`)
  * `resultados/20261107-214223-04-sql-vendas-por-dia/` (`18,55 req/s` | `vendas-por-dia mediana: 2.433,5 ms`)
* **Medição (Antes [`03`] → Depois [`04` — Mediana]):**
  * **Rota `vendas-por-dia` (Mediana):** `5.635,6 ms` → **`2.492,9 ms`** (**-55,8%**)
  * **Rota `vendas-por-dia` (p95):** `7.240,8 ms` → **`6.126,3 ms`** (**-15,4%**)
  * **CPU média da API:** `20,0%` → **`3,3%`** (**-83,5%**)
  * **Memória máxima da API:** `92,5 MiB` → **`25,9 MiB`** (**-72,0%**)
  * **Rotas rápidas beneficiadas pela folga no Event Loop da API:** `cliente` caiu de `15,6 ms` para **`3,8 ms`** na mediana, e `cliente-por-email` caiu de `35,0 ms` para **`5,5 ms`**.
* **Decisão:** **Mantida.** Eliminou o gargalo de alocação de memória e bloqueio do *Event Loop* no container da API, reduzindo o tempo do relatório pela metade.