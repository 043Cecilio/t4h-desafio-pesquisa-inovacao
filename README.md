# \# Desafio de Performance e Otimização — Node.js \& PostgreSQL

# 

# Repositório com o diagnóstico, experimentos iterativos e otimizações de performance aplicados a uma API REST em \*\*Node.js 20 (Express)\*\* conectada ao \*\*PostgreSQL 16\*\*, operando sob restrições severas de CPU e memória via Docker Compose.

# 

# \---

# 

# \## Sumário

# 

# 1\. \[Visão Geral e Arquitetura](#1-visão-geral-e-arquitetura)

# 2\. \[Metodologia e Regras Cumpridas](#2-metodologia-e-regras-cumpridas)

# 3\. \[Diagnóstico Inicial e Gargalos Identificados](#3-diagnóstico-inicial-e-gargalos-identificados)

# 4\. \[Ciclo de Experimentos e Intervenções](#4-ciclo-de-experimentos-e-intervenções)

# 5\. \[Resultados Comparativos (Antes vs. Depois)](#5-resultados-comparativos-antes-vs-depois)

# 6\. \[Estrutura do Repositório](#6-estrutura-do-repositório)

# 7\. \[Como Executar e Reproduzir os Testes](#7-como-executar-e-reproduzir-os-testes)

# 

# \---

# 

# \## 1. Visão Geral e Arquitetura

# 

# A aplicação simula o backend de uma loja virtual com \*\*2.120.000 registros\*\* gerados deterministicamente na inicialização do banco de dados (`banco/02-dados.sql`):

# 

# | Tabela | Volume de Dados | Descrição |

# | :--- | :---: | :--- |

# | `clientes` | 100.000 linhas | Cadastro de clientes (`id`, `nome`, `email`, `cidade`, `criado\_em`) |

# | `produtos` | 20.000 linhas | Catálogo de produtos (`id`, `nome`, `categoria`, `preco\_centavos`, `ativo`) |

# | `pedidos` | 500.000 linhas | Cabeçalho de pedidos (`id`, `cliente\_id`, `status`, `criado\_em`, `observacoes`) |

# | `itens\_pedido` | 1.500.000 linhas | Itens de cada pedido (`id`, `pedido\_id`, `produto\_id`, `quantidade`, `preco\_unitario\_centavos`) |

# 

# \### Limites de Recursos dos Containers (`docker-compose.yml`)

# \* \*\*Banco de Dados (`postgres:16`):\*\* Limite de \*\*1,0 CPU\*\* (`cpus=1000000000`) e \*\*512 MiB de RAM\*\* (`536870912 bytes`).

# \* \*\*API (`node:20-alpine`):\*\* Limite de \*\*0,5 CPU\*\* (`cpus=500000000`) e \*\*256 MiB de RAM\*\* (`268435456 bytes`).

# \* \*\*Perfil de Carga (`k6`):\*\* 20 usuários virtuais simultâneos (`VUS=20`) durante `60s`, precedidos por `15s` de aquecimento (`AQUECIMENTO=15s`), com semente determinística (`SEMENTE=42`).

# 

# \---

# 

# \## 2. Metodologia e Regras Cumpridas

# 

# \* \*\*Verificação de Contrato (`contrato/`):\*\* O comportamento original da API foi gravado em `contrato/base.json` (42 casos de teste cobrindo rotas válidas, paginação, ordenação, filtros e erros HTTP 400/404). Todas as otimizações mantiveram \*\*100% de compatibilidade\*\* (`Contrato mantido: as 42 respostas são iguais às gravadas`).

# \* \*\*Imutabilidade dos Arquivos Base:\*\* Os diretórios `carga/`, `scripts/` e os arquivos `banco/01-estrutura.sql` e `banco/02-dados.sql` não foram alterados. Os novos índices foram adicionados em `banco/03-indices.sql`.

# \* \*\*Isolamento de Variáveis e Repetibilidade:\*\* Cada hipótese foi implementada em commit próprio antes da medição (`arquivos alterados sem commit: 0` em `contexto.txt`) e avaliada pela \*\*mediana de 3 execuções completas\*\* do script `scripts/medir.sh`.

# 

# \---

# 

# \## 3. Diagnóstico Inicial e Gargalos Identificados

# 

# As evidências coletadas após a bateria de linha de base estão salvas em `diagnostico/01-estatisticas-e-indices-iniciais.txt` e `diagnostico/02-explain-analyze-linha-de-base.txt`. Os quatro problemas centrais encontrados foram:

# 

# 1\. \*\*Abertura e Fechamento de Conexão a Cada Query (`api/src/db.js`):\*\*

# &#x20;  A função `consultar()` instanciava um novo `pg.Client()`, realizava \*handshake\* TCP, autenticava no PostgreSQL (criando um processo \*backend\* dedicado) e encerrava a conexão ao final de cada consulta SQL.

# 2\. \*\*Ausência de Índices em Chaves Estrangeiras e Colunas de Filtro (`pg\_stat\_user\_tables`):\*\*

# &#x20;  O esquema original continha apenas os 4 índices de chave primária (`clientes\_pkey`, `pedidos\_pkey`, `itens\_pedido\_pkey`, `produtos\_pkey`). Na linha de base:

# &#x20;  \* `itens\_pedido` sofreu \*\*2.579 \*Sequential Scans\*\*\*, lendo \*\*1.960.500.000 de tuplas\*\* e \*\*0 buscas por índice\*\* (`idx\_scan = 0`).

# &#x20;  \* `pedidos` sofreu \*\*1.287 \*Sequential Scans\*\*\*, lendo \*\*325.000.000 de tuplas\*\*.

# &#x20;  \* `clientes` sofreu \*\*170 \*Sequential Scans\*\*\* (16.900.000 tuplas lidas) na busca por `lower(email)`.

# 3\. \*\*Antipadrão de Consultas N+1 (`GET /clientes/:id/pedidos`):\*\*

# &#x20;  Para retornar os pedidos de um cliente, a rota executava 1 consulta em `clientes`, 1 consulta em `pedidos` (\~336 ms em \*Parallel Seq Scan\*), seguida por um laço `for` com 1 consulta em `itens\_pedido` para cada pedido (\~115,6 ms cada) e outro laço `for` buscando o nome do produto em `produtos` para cada item.

# 4\. \*\*Filtro Não-SARGable e Agregação em Memória no Node.js (`GET /relatorios/vendas-por-dia`):\*\*

# &#x20;  A consulta aplicava `p.criado\_em::date BETWEEN $1 AND $2`, impedindo o uso de índice B-Tree sobre `criado\_em`, além de selecionar `p.observacoes` (coluna larga de texto) e transferir cerca de \*\*12.945 linhas brutas por requisição\*\* para o Node.js agrupar em memória com `Map` e `Set`.

# 

# \---

# 

# \## 4. Ciclo de Experimentos e Intervenções

# 

# \### Experimento 00 — Linha de Base (`00-linha-de-base`)

# \* \*\*Ação:\*\* Medição da aplicação original sem alterações.

# \* \*\*Resultado (Mediana):\*\* \*\*4,41 req/s\*\*, latência mediana de \*\*691,1 ms\*\*, p95 geral de \*\*15.274,7 ms\*\* e CPU do PostgreSQL saturada em \*\*98,2%\*\*.

# 

# \### Experimento 01 — Pool de Conexões (`01-pool-conexoes`)

# \* \*\*Hipótese:\*\* Substituir `pg.Client` por `pg.Pool` (`max: 20`) em `api/src/db.js` elimina o \*overhead\* de abertura de conexão a cada query.

# \* \*\*Resultado:\*\* Vazão subiu para \*\*5,50 req/s (+24,7%)\*\*, latência mediana geral caiu para \*\*451,4 ms (-34,7%)\*\* e p95 geral caiu para \*\*12.662,6 ms (-17,1%)\*\*. Na rota leve `GET /clientes/:id`, a mediana caiu de \*\*292,4 ms para 98,4 ms\*\*.

# 

# \### Experimento 02 — Índices B-Tree no PostgreSQL (`02-indices`)

# \* \*\*Hipótese:\*\* Criar índices nas chaves estrangeiras e colunas filtradas (`banco/03-indices.sql`: `clientes(lower(email))`, `pedidos(cliente\_id)`, `pedidos(status)`, `pedidos(criado\_em)`, `itens\_pedido(pedido\_id)`, `itens\_pedido(produto\_id)`) elimina os \*Sequential Scans\* em consultas pontuais.

# \* \*\*Resultado:\*\* Salto de vazão para \*\*18,54 req/s (+320,4% sobre a base)\*\*, queda da latência mediana geral para \*\*101,4 ms (-85,3%)\*\* e redução do p95 geral para \*\*6.416,5 ms\*\*. A mediana da rota `pedidos-do-cliente` despencou de \*\*9.183,7 ms para 291,8 ms\*\*.

# 

# \### Experimento 03 — Eliminação de N+1 com `LEFT JOIN` (`03-join-pedidos-cliente`)

# \* \*\*Hipótese:\*\* Substituir os laços de consultas N+1 em `GET /clientes/:id/pedidos` por uma única query com `LEFT JOIN` entre `pedidos`, `itens\_pedido` e `produtos` reduz o tempo de resposta da rota e o número de \*round-trips\* ao banco.

# \* \*\*Resultado (Mediana):\*\* A mediana da rota `pedidos-do-cliente` caiu de \*\*291,8 ms para 95,1 ms (-67,4%)\*\* e o p95 da rota caiu de \*\*709,9 ms para 290,2 ms (-59,1%)\*\*. A vazão geral atingiu \*\*19,37 req/s\*\* (pico de 20,15 req/s).

# 

# \### Experimento 04 — Predicado SARGable e Agregação no SQL (`04-sql-vendas-por-dia`)

# \* \*\*Hipótese:\*\* Remover o \*cast\* `::date` da coluna `p.criado\_em` (`p.criado\_em >= $1::timestamp AND p.criado\_em < ($2::date + INTERVAL '1 day')`) e delegar o `GROUP BY`, `COUNT(DISTINCT p.id)` e `SUM()` ao PostgreSQL reduz a latência do relatório e o consumo de CPU/RAM da API.

# \* \*\*Resultado (Mediana):\*\* A mediana da rota `vendas-por-dia` caiu de \*\*5.635,6 ms para 2.492,9 ms (-55,8%)\*\* e o p95 caiu de \*\*7.240,8 ms para 6.126,3 ms\*\*. Na API Node.js, a CPU média despencou de \*\*20,0% para 3,3% (-83,5%)\*\* e o pico de memória RAM caiu de \*\*92,5 MiB para 25,9 MiB (-72,0%)\*\*.

# 

# \---

# 

# \## 5. Resultados Comparativos (Antes vs. Depois)

# 

# \### Métricas Gerais do Sistema (Mediana de 3 Execuções)

# 

# | Experimento | Requisições (60s) | Vazão (req/s) | Mediana Geral | p95 Geral | p99 Geral | CPU Média Banco | CPU Média API | Memória Máx API |

# | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |

# | \*\*`00-linha-de-base`\*\* | 290 | 4,41 | 691,1 ms | 15.274,7 ms | 19.311,1 ms | 98,2% | 8,3% | 45,2 MiB |

# | \*\*`01-pool-conexoes`\*\* | 350 | 5,50 | 451,4 ms | 12.662,6 ms | 16.915,0 ms | 98,4% | 6,3% | 48,0 MiB |

# | \*\*`02-indices`\*\* | 1.320 | 18,54 | 101,4 ms | 6.416,5 ms | 13.344,9 ms | 99,8% | 20,4% | 90,7 MiB |

# | \*\*`03-join-pedidos-cliente`\*\* | 1.317 | \*\*19,37\*\* | 95,4 ms | \*\*6.233,7 ms\*\* | \*\*13.101,3 ms\*\* | 99,5% | 20,0% | 92,5 MiB |

# | \*\*`04-sql-vendas-por-dia`\*\* | 1.244 | 18,55 | \*\*92,5 ms\*\* | 6.965,5 ms | 16.868,8 ms | 100,3% | \*\*3,3%\*\* | \*\*25,9 MiB\*\* |

# 

# \### Desempenho por Rota — Linha de Base (`00`) vs. Estado Final (`04`)

# 

# | Rota | Endpoint | Mediana Antes (`00`) | Mediana Depois (`04`) | p95 Antes (`00`) | p95 Depois (`04`) | Evolução (Mediana) |

# | :--- | :--- | :---: | :---: | :---: | :---: | :---: |

# | `cliente` | `GET /clientes/:id` | 292,4 ms | \*\*3,8 ms\*\* | 601,6 ms | \*\*101,7 ms\*\* | \*\*-98,7%\*\* |

# | `cliente-por-email` | `GET /clientes?email=` | 519,8 ms | \*\*5,5 ms\*\* | 2.230,8 ms | \*\*120,6 ms\*\* | \*\*-98,9%\*\* |

# | `pedidos-do-cliente` | `GET /clientes/:id/pedidos` | 11.913,6 ms | \*\*92,9 ms\*\* | 19.126,2 ms | \*\*202,7 ms\*\* | \*\*-99,2%\*\* |

# | `busca-produtos` | `GET /produtos/busca?q=` | 394,3 ms | \*\*96,7 ms\*\* | 743,3 ms | \*\*202,8 ms\*\* | \*\*-75,5%\*\* |

# | `pedidos-por-status` | `GET /pedidos?status=\&pagina=` | 7.385,1 ms | \*\*595,3 ms\*\* | 13.661,5 ms | \*\*12.578,5 ms\*\* | \*\*-91,9%\*\* |

# | `vendas-por-dia` | `GET /relatorios/vendas-por-dia` | 4.421,0 ms | \*\*2.492,9 ms\*\* | 7.518,2 ms | \*\*6.126,3 ms\*\* | \*\*-43,6%\*\* |

# | `top-produtos` | `GET /relatorios/top-produtos` | 12.760,4 ms | \*\*16.813,3 ms\*\* | 18.817,2 ms | \*\*32.301,8 ms\*\* | \*Ver `proximos-testes.md`\* |

# 

# \---

# 

# \## 6. Estrutura do Repositório

# 

# \* \*\*`api/src/db.js`\*\*: Configuração do pool de conexões (`pg.Pool`) com o PostgreSQL.

# \* \*\*`api/src/rotas.js`\*\*: Implementação das rotas HTTP otimizadas (`LEFT JOIN` e agregação SQL).

# \* \*\*`banco/01-estrutura.sql` \& `02-dados.sql`\*\*: Estrutura e carga de dados originais (intocados).

# \* \*\*`banco/03-indices.sql`\*\*: Criação dos índices B-Tree secundários e funcionais.

# \* \*\*`contrato/`\*\*: Base de respostas original (`base.json`) e verificador de regressão (`verificar.js`).

# \* \*\*`diagnostico/`\*\*: Planos `EXPLAIN (ANALYZE, BUFFERS)` e estatísticas `pg\_stat\_user\_tables` da linha de base.

# \* \*\*`resultados/`\*\*: Diretórios gerados pelo `scripts/medir.sh` contendo `contexto.txt`, `resumo.txt`, `resumo-k6.json`, `recursos.txt`, `recursos.csv` e `contrato.txt`.

# \* \*\*`experimentos.md`\*\*: Registro detalhado das hipóteses, mudanças e métricas de cada ciclo.

# \* \*\*`proximos-testes.md`\*\*: Plano priorizado para próximos ciclos (visões materializadas para `top-produtos`, índices compostos de cobertura com `INCLUDE` para paginação e `pg\_trgm`).

# \* \*\*`artigo/artigo.md`\*\*: Artigo técnico narrando a investigação, evidências e conclusões de engenharia.

# 

# \---

# 

# \## 7. Como Executar e Reproduzir os Testes

# 

# \### Pré-requisitos

# \* \*\*Docker\*\* e \*\*Docker Compose\*\* instalados.

# \* Se estiver utilizando \*\*Git Bash no Windows\*\*, desative a conversão automática de caminhos antes de rodar os scripts:

# &#x20; ```bash

# &#x20; export MSYS\_NO\_PATHCONV=1

# &#x20; ```

# 

# \### 1. Subir o ambiente do zero (recriando o volume do banco)

# Os scripts em `banco/` são executados pelo PostgreSQL apenas na criação de um volume limpo:

# 

# ```bash

# docker compose down -v

# docker compose up -d --build

# ```

# 

# \### 2. Verificar a saúde da API e o contrato

# ```bash

# curl http://localhost:3000/saude

# scripts/verificar-contrato.sh comparar

# ```

# 

# \### 3. Executar uma medição de carga

# ```bash

# scripts/medir.sh meu-teste

# ```

# Os resultados detalhados serão gerados automaticamente em `resultados/<timestamp>-meu-teste/`.

