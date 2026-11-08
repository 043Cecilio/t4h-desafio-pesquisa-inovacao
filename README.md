# Desafio de Performance e Otimização — Estágio em Pesquisa e Inovação 2026

Todos os comandos, do zero até a linha de base e às medições de otimização, estão reunidos neste `README.md`. Ele foi estruturado para mostrar o comando exato e a resposta esperada em cada etapa. Você só precisa de um computador com cerca de 2 GB livres: **Node.js 20**, **PostgreSQL 16** e **k6** rodam inteiramente em containers, sem necessidade de instalar nada além do **Docker** e do **Git**.

---

## Sumário

1. [Passo a passo do zero](#1-passo-a-passo-do-zero)
2. [Gravar o contrato e medir a linha de base](#2-gravar-o-contrato-e-medir-a-linha-de-base)
3. [Como medir](#3-como-medir)
4. [Acessar o banco](#4-acessar-o-banco)
5. [Problemas comuns](#5-problemas-comuns)
6. [Resumo do Desafio: Problema Diagnosticado e Resultados Obtidos](#6-resumo-do-desafio-problema-diagnosticado-e-resultados-obtidos)

---

## 1. Passo a passo do zero

### 1.1. Abrir o terminal e verificar Git e Docker
Abra o terminal (no Windows, utilize o **Git Bash**) e confirme que o Git e o Docker estão instalados e rodando:

```bash
git --version
docker --version
docker compose version
```

**Resposta esperada:**
```text
git version 2.x.x
Docker version 27.x.x, build ...
Docker Compose version v2.x.x
```

> **Importante para usuários Windows (Git Bash):** Desative a conversão automática de barras de caminho POSIX antes de rodar os scripts do projeto:
> ```bash
> export MSYS_NO_PATHCONV=1
> ```

### 1.2. Copiar a pasta do desafio para um repositório seu
Clone o repositório de desafios, copie a pasta correspondente para uma pasta própria e inicialize o seu repositório Git:

```bash
git clone [https://github.com/tech4humans-brasil/t4h-desafios.git](https://github.com/tech4humans-brasil/t4h-desafios.git)
cp -R "t4h-desafios/Vaga - Estágio Pesquisa e Inovação 2026" meu-desafio
cd meu-desafio
git init
git add .
git commit -m "chore: estado inicial do desafio"
```

Conecte ao seu repositório público no GitHub:
```bash
git branch -M main
git remote add origin [https://github.com/043Cecilio/t4h-desafio-pesquisa-inovacao.git](https://github.com/043Cecilio/t4h-desafio-pesquisa-inovacao.git)
git push -u origin main
```

### 1.3. Subir a aplicação
Construa as imagens e suba os containers em segundo plano:

```bash
docker compose up -d --build
```

**Resposta esperada:**
```text
 ✔ Network desafio-performance_default    Created
 ✔ Volume "desafio-performance_dados-banco" Created
 ✔ Container desafio-performance-banco-1  Healthy
 ✔ Container desafio-performance-api-1    Started
```
*(Na primeira execução, o PostgreSQL criará as tabelas de `banco/01-estrutura.sql` e gerará os 2.120.000 registros de `banco/02-dados.sql`, o que leva cerca de 20 a 40 segundos até o container `banco` ficar `Healthy`).*

### 1.4. Fazer as primeiras chamadas
Teste se a API está no ar e respondendo às rotas:

```bash
curl http://localhost:3000/saude
```
**Resposta esperada:**
```json
{"status":"ok"}
```

Consulte um cliente pelo ID:
```bash
curl http://localhost:3000/clientes/1
```
**Resposta esperada:**
```json
{"id":1,"nome":"Cliente 1","email":"cliente1@exemplo.com","cidade":"São Paulo","criado_em":"..."}
```

---

## 2. Gravar o contrato e medir a linha de base

> **Regra de ouro:** Antes de mudar qualquer coisa, grave o contrato e meça a linha de base três vezes, com a aplicação exatamente como veio. Se fizer isso depois de mexer no código, nenhuma comparação vale.

### 2.1. Gravar o contrato original (`contrato/base.json`)
```bash
scripts/verificar-contrato.sh gravar
```

**Resposta esperada:**
```text
Contrato gravado em contrato/base.json (42 respostas).
```

Para verificar a qualquer momento se as 42 respostas continuam idênticas às gravadas:
```bash
scripts/verificar-contrato.sh comparar
```

**Resposta esperada:**
```text
igual     /clientes/1
...
igual     /relatorios/top-produtos?limite=51

Contrato mantido: as 42 respostas são iguais às gravadas.
```

### 2.2. Medir a linha de base 3 vezes
Com o código original intacto, executam-se três medições seguidas:

```bash
scripts/medir.sh 00-linha-de-base
scripts/medir.sh 00-linha-de-base
scripts/medir.sh 00-linha-de-base
```

**Resposta esperada ao final de cada execução:**
```text
Aquecendo por 15s (não entra na medição)...
Medindo: 20 usuários virtuais por 60s...
Medição: 00-linha-de-base
Carga: 20 usuários virtuais por 60s

Requisições:       290
Requisições/s:     4.41
Tempo de resposta: mediana 691.1 ms | p95 14901.2 ms | p99 18493.4 ms
Erros:             0.00%
...
Contrato: Contrato mantido: as 42 respostas são iguais às gravadas.
Resultado salvo em resultados/20261107-173932-00-linha-de-base/
```

---

## 3. Como medir

### 3.1. O que o script `scripts/medir.sh` faz
Ao rodar `scripts/medir.sh <rotulo>`:
1. Verifica se a API está respondendo em `http://127.0.0.1:3000/saude`.
2. Registra os dados do ambiente, commit atual, quantidade de arquivos sem commit e limites de hardware.
3. Executa um aquecimento de 15s com 5 usuários virtuais e semente `43` (não contabilizado).
4. Coleta amostras de CPU e RAM dos containers em background com `docker stats`.
5. Executa a carga principal de 60s com **20 usuários virtuais** (`VUS=20`) e semente fixa (`SEMENTE=42`) usando o k6.
6. Roda automaticamente o verificador de contrato (`contrato comparar`) após a carga.

### 3.2. O que vai em cada arquivo de `resultados/<data>-<rotulo>/`
* **`resumo.txt`**: Vazão (req/s), total de requisições, erros e tempos (`mediana`, `p95`, `p99`) gerais e por rota.
* **`resumo-k6.json`**: Métricas brutas completas exportadas pelo k6.
* **`recursos.txt`**: Tabela com consumo médio e máximo de CPU e RAM dos containers `banco` e `api`.
* **`recursos.csv`**: Série temporal segundo a segundo coletada pelo `docker stats`.
* **`contexto.txt`**: Metadados da execução, commit, arquivos sem commit e limites do Docker.
* **`contrato.txt`**: Saída completa das 42 verificações de contrato pós-carga.

### 3.3. Como reencontrar as medições e medir de novo
Para listar e consultar os resumos:
```bash
ls -la resultados/
cat resultados/*00-linha-de-base/resumo.txt
```
Para refazer medições anteriores, basta garantir que o repositório esteja no commit correspondente, recriar o volume do banco se necessário (`docker compose down -v`) e rodar o `scripts/medir.sh`.

---

## 4. Acessar o banco

### 4.1. Conectar no PostgreSQL pelo terminal (`psql`)
```bash
docker compose exec banco psql -U loja -d loja
```
Comandos úteis: `\dt`, `\di`, `EXPLAIN (ANALYZE, BUFFERS) SELECT ...;`, `\q`.

### 4.2. Conectar por DBeaver ou pgAdmin
* **Host:** `localhost` | **Porta:** `5432` | **Database:** `loja` | **User:** `loja` | **Pass:** `loja`

### 4.3. Como mudar parâmetros de forma reprodutível
1. Não edite `banco/01-estrutura.sql` nem `banco/02-dados.sql`.
2. Crie arquivos em ordem alfabética a partir de `03-` em `banco/` (ex.: `banco/03-indices.sql`).
3. Recrie o volume do banco para aplicar:
   ```bash
   docker compose down -v
   docker compose up -d --build
   ```

---

## 5. Problemas comuns

| Problema / Sintoma | Causa | Como resolver |
| :--- | :--- | :--- |
| **`A API não respondeu em http://127.0.0.1:3000/saude`** | Containers parados ou PostgreSQL populando os dados iniciais. | Aguarde o container ficar `Healthy` e rode `docker compose up -d --build`. |
| **O script `medir.sh` para no aquecimento no Windows (Git Bash)** | Conversão automática de caminhos POSIX pelo Git Bash. | Execute `export MSYS_NO_PATHCONV=1` antes de rodar `scripts/medir.sh`. |
| **Criei o arquivo em `banco/`, mas os índices não aparecem** | Scripts em `docker-entrypoint-initdb.d/` só rodam na criação inicial do volume. | Rode `docker compose down -v` e depois `docker compose up -d --build`. |
| **`arquivos alterados sem commit` maior que 0 em `contexto.txt`** | O script foi rodado com arquivos pendentes no Git. | Faça `git add` e `git commit` das alterações **antes** de medir. |

---

## 6. Resumo do Desafio: Problema Diagnosticado e Resultados Obtidos

### O Problema
Em sua configuração original (Node.js 20 com 0,5 CPU / 256 MiB de RAM e PostgreSQL 16 com 1,0 CPU / 512 MiB de RAM, sobre 2,12 milhões de registros), a aplicação suportava apenas **4,41 req/s** na mediana, com latência p95 geral de **15.274,7 ms** e CPU do banco saturada em **98,2%**. O diagnóstico (`diagnostico/`) identificou quatro causas principais:
1. Abertura e fechamento de conexão TCP (`pg.Client`) a cada query executada (`api/src/db.js`).
2. Ausência total de índices secundários e em chaves estrangeiras, gerando **1,96 bilhão de tuplas lidas em *Sequential Scans*** em `itens_pedido` e **325 milhões** em `pedidos`.
3. Consultas N+1 sequenciais na rota `GET /clientes/:id/pedidos` (`api/src/rotas.js`).
4. Filtro de data não-SARGable (`criado_em::date`) e agregação em memória no Node.js na rota `GET /relatorios/vendas-por-dia`.

### Resumo dos Resultados (Mediana de 3 Execuções no `scripts/medir.sh`)

| Etapa / Rótulo | Vazão (req/s) | Mediana Geral | p95 Geral | CPU Média Banco | CPU Média API | Memória Máx API | Contrato (42 respostas) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`00-linha-de-base`** | 4,41 | 691,1 ms | 15.274,7 ms | 98,2% | 8,3% | 45,2 MiB | Mantido |
| **`01-pool-conexoes`** | 5,50 (+24,7%) | 451,4 ms (-34,7%) | 12.662,6 ms (-17,1%) | 98,4% | 6,3% | 48,0 MiB | Mantido |
| **`02-indices`** | 18,54 (+320,4%) | 101,4 ms (-85,3%) | 6.416,5 ms (-58,0%) | 99,8% | 20,4% | 90,7 MiB | Mantido |
| **`03-join-pedidos-cliente`** | 19,37 (+339,2%) | 95,4 ms (-86,2%) | 6.233,7 ms (-59,2%) | 99,5% | 20,0% | 92,5 MiB | Mantido |
| **`04-sql-vendas-por-dia`** | 18,55 (+320,6%) | 92,5 ms (-86,6%) | 6.965,5 ms (-54,4%) | 100,3% | 3,3% | 25,9 MiB | Mantido |

* **Registro detalhado dos experimentos:** [experimentos.md](experimentos.md)
* **Fichas de hipóteses para testes futuros:** [proximos-testes.md](proximos-testes.md)
* **Artigo técnico completo:** [artigo/artigo.md](artigo/artigo.md)