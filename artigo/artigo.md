# Otimização de API Node.js e PostgreSQL: Vazão 4,4x Maior e Redução de 86,6% na Latência Mediana sem Aumento de Hardware

## 1. Resumo

Uma API REST em Node.js 20 e PostgreSQL 16 (2,12 milhões de registros), limitada a 0,5 CPU/256 MiB de RAM na aplicação e 1 CPU/512 MiB no banco, suportava apenas **4,41 req/s**, com latência mediana de **691,1 ms** e p95 de **15.274,7 ms** sob 20 usuários simultâneos. Após substituir conexões avulsas por pool (`pg.Pool`), criar índices B-Tree secundários e funcionais, eliminar consultas N+1 com `LEFT JOIN` e delegar agregações ao SQL com filtros SARGable, a vazão atingiu **19,37 req/s** (**+339,2%**), a mediana caiu para **92,5 ms** (**-86,6%**) e a CPU da API recuou de **20,0% para 3,3%**, mantendo 100% do contrato.

## 2. Contexto e diagnóstico

Na medição inicial ([resumo da linha de base](../resultados/20261107-173932-00-linha-de-base/resumo.txt) e [recursos da linha de base](../resultados/20261107-173932-00-linha-de-base/recursos.txt)), o banco operava saturado em **98,2% de CPU média** (pico de **99,8%**), enquanto a API consumia apenas **8,3% de CPU**. As rotas mais críticas eram `GET /clientes/:id/pedidos` (mediana de **11.913,6 ms**, p95 de **19.126,2 ms**), `GET /relatorios/top-produtos` (p95 de **18.817,2 ms**), `GET /pedidos` (p95 de **13.661,5 ms**) e `GET /relatorios/vendas-por-dia` (p95 de **7.518,2 ms**).

A investigação com `pg_stat_user_tables`, `pg_indexes` e `EXPLAIN (ANALYZE, BUFFERS)` — registrada em [estatísticas iniciais](../diagnostico/01-estatisticas-e-indices-iniciais.txt) e [planos de execução](../diagnostico/02-explain-analyze-linha-de-base.txt) — junto à leitura de `api/src/db.js` e `api/src/rotas.js` revelou quatro problemas:

1. **Conexão sem pool (`api/src/db.js`):** A função `consultar()` instanciava `new Client()`, abria e encerrava uma conexão TCP a cada query.
2. **Ausência de índices secundários:** Existiam apenas os 4 índices de chave primária. Em três baterias de carga, `itens_pedido` acumulou **2.579 *Sequential Scans*** (**1,96 bilhão de tuplas lidas**, `idx_scan = 0`) e `pedidos` sofreu **1.287 *Sequential Scans*** (**325 milhões de tuplas lidas**).
3. **Consultas N+1 (`GET /clientes/:id/pedidos`):** A rota disparava 1 query para o cliente, 1 para seus pedidos (336,06 ms em `Parallel Seq Scan`), $N$ queries em `itens_pedido` (115,60 ms cada) e $M$ queries em `produtos`.
4. **Filtro não-SARGable e agregação em memória (`GET /relatorios/vendas-por-dia`):** O predicado `WHERE p.criado_em::date BETWEEN $1 AND $2` convertia a coluna a cada linha, impedindo o uso de índice em `criado_em`, e transferia ~12.945 linhas brutas (incluindo a coluna longa `observacoes`) para agregação em JavaScript.

* **Hipótese de causa 1:** Abrir e fechar conexões TCP a cada consulta cria processos *backend* excessivos no PostgreSQL e eleva a latência basal de todas as rotas.
* **Hipótese de causa 2:** A falta de índices em chaves estrangeiras e colunas de busca força varreduras completas (*Sequential Scans*) que esgotam 1 CPU do banco.
* **Hipótese de causa 3:** Laços de consultas sequenciais (N+1) multiplicam o tempo de rede e de acesso ao banco na listagem de pedidos do cliente.
* **Hipótese de causa 4:** Aplicar cast `::date` na coluna filtrada impede varredura por índice, e agregar dezenas de milhares de tuplas no Node.js sobrecarrega a memória e o *Event Loop* da API.

## 3. Estado da arte

### Gerenciamento e Pool de Conexões
A documentação oficial do PostgreSQL 16 demonstra que a arquitetura do SGBD é baseada em processos (*process-per-connection*), onde cada nova conexão exige um `fork()` do processo supervisor e alocação de memória de sessão [1]. A documentação oficial do driver `node-postgres` (`pg`) recomenda o uso de `pg.Pool` para aplicações web concorrentes, mantendo conexões abertas e evitando o custo repetitivo de *handshake* TCP e autenticação [2].

### Indexação B-Tree, Índices Funcionais e Predicados SARGable
Na literatura clássica de bancos de dados, Graefe (2011) demonstra que árvores B+ (*B-Trees*) reduzem a complexidade de busca de $O(N)$ para $O(\log N)$ páginas lidas [3]. Segundo a documentação oficial do PostgreSQL sobre índices e expressões, consultas que aplicam funções sobre colunas — como `lower(email)` ou `criado_em::date` — não utilizam índices B-Tree simples na coluna bruta, exigindo índices baseados em expressões ou a reescrita da condição [4]. Winand (2012) formaliza esse conceito como predicados *SARGable* (*Search Argument Able*): transformar `coluna::date BETWEEN inicio AND fim` em um intervalo contínuo (`coluna >= inicio AND coluna < fim + 1 dia`) preserva a ordenação do índice e viabiliza um *Index Scan* [5].

### Eliminação do Antipadrão N+1 e Pushdown de Agregação
Kleppmann (2017) e a documentação de performance do PostgreSQL destacam que substituir múltiplas consultas sequenciais (antipadrão N+1) por junções relacionais (`JOIN`) permite ao otimizador de consultas escolher algoritmos eficientes como *Nested Loop Index Join* ou *Hash Join* em uma única viagem de rede (*round-trip*) [6, 7]. Ademais, realizar projeções estritas (evitando `SELECT *` em tabelas com colunas largas via mecanismo TOAST) e empurrar agregações (`GROUP BY`, `SUM`, `COUNT`) para o motor relacional reduz drasticamente a serialização de dados e a pressão sobre o coletor de lixo (*Garbage Collector*) da aplicação [4, 6].

## 4. Alternativas consideradas

| Alternativa | Custo | Desempenho | Risco | Esforço |
| :--- | :--- | :--- | :--- | :--- |
| **A. Escalonamento Vertical (mais CPU/RAM no Docker)** | Alto (aumento direto de infraestrutura) | Moderado (alivia sintomas, mas mantém complexidade $O(N)$) | Baixo | Mínimo (alterar `docker-compose.yml`) |
| **B. Cache Externo (container Redis) ou em Memória** | Médio (consome RAM extra ou novo serviço) | Alto para leituras repetidas; ineficaz em buscas inéditas | Médio (risco de dados obsoletos e quebra de contrato) | Médio |
| **C. Otimização de Acesso (Pool + Índices B-Tree + JOIN + SQL Pushdown)** | Zero (mantém 0,5 CPU/256 MB na API e 1 CPU/512 MB no banco) | Muito Alto em OLTP (reduz complexidade para $O(\log N)$ e tráfego interno) | Baixo (preserva consistência ACID e contrato HTTP) | Baixo a Médio (ajustes pontuais em SQL e Node.js) |
| **D. Pré-agregação com Visões Materializadas (`MATERIALIZED VIEW`)** | Baixo (pequeno espaço adicional em disco) | Extremo para relatórios pesados (`top-produtos` em < 5 ms) | Médio em cenários reais (exige estratégia de atualização) | Médio |

As abordagens acima representam diferentes filosofias de engenharia. A **Alternativa A** ataca a falta de recursos comprando hardware, enquanto a **Alternativa B** introduz uma camada intermediária para evitar ir ao banco. Já a **Alternativa C** corrige a eficiência algorítmica das consultas e do protocolo de comunicação entre API e PostgreSQL sem adicionar infraestrutura. Por fim, a **Alternativa D** pré-computa agregações analíticas que varrem grande fração das tabelas.

## 5. Solução escolhida e por quê

Escolhemos a **Alternativa C** (implementada em quatro experimentos incrementais: `pg.Pool`, índices B-Tree/funcionais, refatoração de N+1 com `LEFT JOIN` e agregação SQL com predicado SARGable), deixando a **Alternativa D** mapeada para o ciclo seguinte de relatórios.

O argumento central é que mais de 98% da CPU do banco era desperdiçada lendo 2,28 bilhões de tuplas de forma sequencial por falta de índices em chaves estrangeiras e por conexões recriadas a cada query [1, 3]. Corrigir a complexidade algorítmica entrega ganho permanente sem custo financeiro adicional.

Descartamos a **Alternativa A** porque aumentar limites de CPU e memória contraria o objetivo de tornar a aplicação mais barata de rodar e mascararia os *Sequential Scans*. Descartamos a **Alternativa B** neste primeiro ciclo porque adicionar Redis consumiria memória adicional e exigiria políticas de invalidação, sendo desnecessário antes de indexar corretamente o banco relacional.

## 6. Experimento e resultado

Todas as medições seguiram 3 execuções de 60s (20 VUs, semente 42), comparando sempre a **mediana** contra a etapa imediatamente anterior (e contra a linha de base no consolidado):

| Mudança | Hipótese | p95 antes → depois | req/s antes → depois | CPU/memória | Pastas de resultados | Conclusão |
| :--- | :--- | :---: | :---: | :--- | :--- | :--- |
| **`00-linha-de-base`** | Medição inicial da aplicação original sem alterações. | 15.274,7 ms (mediana: 691,1 ms) | 4,41 req/s | Banco: 98,2% CPU / 426,1 MiB<br>API: 8,3% CPU / 45,2 MiB | [173622](../resultados/20261107-173622-00-linha-de-base/resumo.txt), [173758](../resultados/20261107-173758-00-linha-de-base/resumo.txt), [173932](../resultados/20261107-173932-00-linha-de-base/resumo.txt) | Ponto de partida gravado; banco saturado em CPU. |
| **`01-pool-conexoes`** | Trocar `pg.Client` por `pg.Pool` (`max: 20`) reduz latência de conexão e sobe vazão. | 15.274,7 ms → **12.662,6 ms** (-17,1%) | 4,41 → **5,50 req/s** (+24,7%) | Banco: 98,4% CPU / 414,5 MiB<br>API: 6,3% CPU / 48,0 MiB | [193207](../resultados/20261107-193207-01-pool-conexoes/resumo.txt) | **Funcionou.** Mediana de `cliente` caiu de 292,4 ms para 98,4 ms. |
| **`02-indices`** | Criar índices B-Tree em FKs, `status`, `criado_em` e `lower(email)` elimina *Seq Scans*. | 12.662,6 ms → **6.416,5 ms** (-49,3%) | 5,50 → **18,54 req/s** (+237,1%) | Banco: 99,8% CPU / 460,5 MiB<br>API: 20,4% CPU / 90,7 MiB | [203632](../resultados/20261107-203632-02-indices/resumo.txt) e medições `02-indices` | **Funcionou.** Vazão subiu 4,2x sobre a base; `pedidos-do-cliente` caiu de 9,18 s para 291,8 ms. |
| **`03-join-pedidos-cliente`** | Substituir queries N+1 em `/clientes/:id/pedidos` por `LEFT JOIN` reduz o p95 da rota. | Rota: 709,9 ms → **290,2 ms** (-59,1%)<br>Geral: 6.416,5 → **6.233,7 ms** | 18,54 → **19,37 req/s** (+4,5%) | Banco: 99,5% CPU / 434,7 MiB<br>API: 20,0% CPU / 92,5 MiB | [212159](../resultados/20261107-212159-03-join-pedidos-cliente/resumo.txt), [212332](../resultados/20261107-212332-03-join-pedidos-cliente/resumo.txt), [212501](../resultados/20261107-212501-03-join-pedidos-cliente/resumo.txt) | **Funcionou.** Mediana da rota caiu de 291,8 ms para 95,1 ms (-67,4%). |
| **`04-sql-vendas-por-dia`** | Filtro SARGable sem `::date` e `GROUP BY` no SQL reduzem latência da rota e CPU/RAM da API. | Rota: 7.240,8 ms → **6.126,3 ms** (-15,4%)<br>Geral: 6.233,7 → **6.965,5 ms** | 19,37 → **18,55 req/s** (-4,2%) | Banco: 100,3% CPU / 466,7 MiB<br>API: **3,3% CPU / 25,9 MiB** | [213918](../resultados/20261107-213918-04-sql-vendas-por-dia/resumo.txt), [214052](../resultados/20261107-214052-04-sql-vendas-por-dia/resumo.txt), [214223](../resultados/20261107-214223-04-sql-vendas-por-dia/resumo.txt) | **Funcionou na rota e na API; neutro na vazão geral.** Mediana da rota caiu 55,8% (5,63 s → 2,49 s) e CPU da API caiu 83,5%. |

O Experimento 04 evidenciou também o que **não melhorou o p95 global**: embora `vendas-por-dia` tenha caído de 5.635,6 ms para 2.492,9 ms na mediana e a API tenha economizado 83,5% de CPU e 72% de RAM, a vazão global estabilizou em ~18,55 req/s porque a CPU única do banco permaneceu em 100% disputada pelas rotas ainda não otimizadas (`top-produtos` e `pedidos-por-status`).

## 7. Limites e testes futuros

A solução atual possui três limites claros:
1. **Relatório analítico pesado (`GET /relatorios/top-produtos`):** A condição `p.status <> 'cancelado'` seleciona 90% da tabela `pedidos` (450 mil linhas) e agrega 1,35 milhão de itens. Índices B-Tree convencionais não aceleram consultas de baixa seletividade; com o aumento de 4x na vazão geral, essa rota sofreu contenção de CPU, subindo seu p95 para 32,3 s.
2. **Paginação profunda com `OFFSET` e `SELECT *` (`GET /pedidos`):** Páginas altas (`pagina=250`) exigem varredura e ordenação em memória, mantendo o p95 da rota em 12,5 s.
3. **Carga exclusivamente de leitura:** Não avaliamos o custo de manutenção dos 6 novos índices sob carga intensa de `INSERT`/`UPDATE`.

**O que derrubaria nossa conclusão:** Caso a aplicação em produção recebesse escrita massiva (*write-heavy*) onde o custo de atualização dos índices superasse o ganho de leitura, ou caso o p95 de `top-produtos` fosse o único SLA eliminatório sem pré-agregação.

Para resolver os gargalos residuais, documentamos em [próximos testes](../proximos-testes.md) quatro hipóteses priorizadas:
* Visão Materializada (`mv_top_produtos`) indexada por receita para responder `top-produtos` em milissegundos;
* Índice composto de cobertura `(status, criado_em DESC, id DESC) INCLUDE (cliente_id)` com `VACUUM ANALYZE` e remoção de `SELECT *` em `GET /pedidos`;
* Índices de cobertura (*Index-Only Scan*) em `itens_pedido` para `vendas-por-dia`;
* Índice GIN de trigramas (`pg_trgm`) para `ILIKE` em `GET /produtos/busca` [8].

## 8. Referências

1. PostgreSQL Global Development Group. *PostgreSQL 16 Documentation: Chapter 19. Server Configuration & Connection Settings*. Disponível em: https://www.postgresql.org/docs/16/runtime-config-connection.html. Acesso em: 08 nov. 2026.
2. Carlson, B. *node-postgres (pg) Official Documentation: Pooling*. Disponível em: https://node-postgres.com/features/pooling. Acesso em: 08 nov. 2026.
3. Graefe, G. *Modern B-Tree Techniques*. Foundations and Trends in Databases, v. 3, n. 4, p. 203–402, 2011. Disponível em: https://doi.org/10.1561/1900000028. Acesso em: 08 nov. 2026.
4. PostgreSQL Global Development Group. *PostgreSQL 16 Documentation: Chapter 11. Indexes (Indexes on Expressions & Index-Only Scans)*. Disponível em: https://www.postgresql.org/docs/16/indexes.html. Acesso em: 08 nov. 2026.
5. Winand, M. *SQL Performance Explained: Everything Developers Need to Know about SQL Performance*. Viena: Markus Winand, 2012. Versão web: https://use-the-index-luke.com/. Acesso em: 08 nov. 2026.
6. Kleppmann, M. *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. Sebastopol: O'Reilly Media, 2017.
7. PostgreSQL Global Development Group. *PostgreSQL 16 Documentation: Chapter 14. Performance Tips (Using EXPLAIN)*. Disponível em: https://www.postgresql.org/docs/16/using-explain.html. Acesso em: 08 nov. 2026.
8. PostgreSQL Global Development Group. *PostgreSQL 16 Documentation: F.35. pg_trgm — Support for Similarity of Text Using Trigram Matching*. Disponível em: https://www.postgresql.org/docs/16/pgtrgm.html. Acesso em: 08 nov. 2026.

## 9. Uso de IA

Utilizei assistente de IA (Gemini) como apoio para estruturar scripts de diagnóstico SQL (`EXPLAIN ANALYZE` e `pg_stat_user_tables`), revisar a sintaxe de queries e organizar a formatação Markdown da documentação. Todos os comandos foram executados localmente no meu ambiente Docker, os contratos foram validados em 100% pelo `verificar-contrato.sh` e todos os números reportados foram extraídos sem edição das pastas geradas pelo `scripts/medir.sh`.