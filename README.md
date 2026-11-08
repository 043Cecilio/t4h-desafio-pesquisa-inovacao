# Desafio de Performance e Otimização — Pesquisa e Inovação 2026

Todos os comandos, do zero até a linha de base e às medições de otimização, estão reunidos neste `README.md`. Ele foi estruturado para mostrar o comando exato e a resposta esperada em cada etapa. Você só precisa de um computador com cerca de 2 GB livres: **Node.js 20**, **PostgreSQL 16** e **k6** rodam inteiramente em containers, sem necessidade de instalar nada além do **Docker** e do **Git**.

---

## Sumário

1. [Passo a passo do zero](#1-passo-a-passo-do-zero)
2. [Gravar o contrato e medir a linha de base](#2-gravar-o-contrato-e-medir-a-linha-de-base)
3. [Como medir](#3-como-medir)
4. [Acessar o banco](#4-acessar-o-banco)
5. [Problemas comuns](#5-problemas-comuns)
6. [Resumo dos Experimentos e Resultados Obtidos](#6-resumo-dos-experimentos-e-resultados-obtidos)

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
Clone o repositório de desafios, copie a pasta `Vaga - Estágio Pesquisa e Inovação 2026` para uma pasta própria e inicialize o seu repositório Git:

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
Para rodar este passo exatamente no estado original (sem os índices adicionados posteriormente), o comando utilizado foi:

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
Com o código original intacto e sem arquivos pendentes de commit, executam-se três medições seguidas:

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
2. Registra os dados do ambiente, commit atual, quantidade de arquivos sem commit e limites de CPU/memória dos containers.
3. Executa um **aquecimento de 15s** (`AQUECIMENTO=15s`) com 5 usuários virtuais e semente `43` (não contabilizado no resultado).
4. Inicia um coletor em background com `docker stats` para amostrar o uso de CPU e RAM de cada serviço.
5. Executa a **carga principal de 60s** (`DURACAO=60s`) com **20 usuários virtuais** (`VUS=20`) e semente fixa (`SEMENTE=42`) usando o k6 (`/carga/teste-carga.js`).
6. Roda automaticamente o verificador de contrato (`contrato comparar`) após a carga para garantir que nada quebrou.

### 3.2. O que vai em cada arquivo de `resultados/<data>-<rotulo>/`
Cada execução cria uma pasta em `resultados/` contendo 6 arquivos:
* **`resumo.txt`**: Vazão (requisições/s), total de requisições, taxa de erros e tempos de resposta (`mediana`, `p95`, `p99`) gerais e discriminados pelas 7 rotas.
* **`resumo-k6.json`**: Métricas brutas completas exportadas pelo k6 em formato JSON.
* **`recursos.txt`**: Tabela consolidada com número de amostras, `cpu_media`, `cpu_max`, `mem_media` e `mem_max` dos containers `banco` e `api`.
* **`recursos.csv`**: Série temporal bruta coletada pelo `docker stats` segundo a segundo durante a carga.
* **`contexto.txt`**: Metadados da execução (rótulo, data, VUs, duração, semente, hash do commit, `arquivos alterados sem commit` e limites de hardware do Docker).
* **`contrato.txt`**: Saída completa das 42 verificações de contrato realizadas logo após a carga.

### 3.3. Como reencontrar as medições
Para listar todas as pastas de medição salvas e consultar os resumos de um experimento específico:

```bash
ls -la resultados/
cat resultados/*00-linha-de-base/resumo.txt
cat resultados/*04-sql-vendas-por-dia/resumo.txt
```

### 3.4. Como medir o "antes" de novo
Caso queira reproduzir o estado inicial ("antes") para comparar novamente do zero:
1. Faça checkout do commit inicial (ou mova temporariamente os arquivos a partir de `banco/03-*.sql` e reverta `api/src/`):
   ```bash
   git checkout 3538269
   ```
2. Recrie o volume do banco e reconstrua a API:
   ```bash
   docker compose down -v
   docker compose up -d --build
   ```
3. Para voltar à versão final otimizada:
   ```bash
   git checkout main
   docker compose down -v
   docker compose up -d --build
   ```

---

## 4. Acessar o banco

### 4.1. Conectar no PostgreSQL pelo terminal (`psql`)
Com os containers rodando, abra um terminal interativo do PostgreSQL:

```bash
docker compose exec banco psql -U loja -d loja
```

Comandos úteis dentro do `psql`:
* `\dt` — Lista as tabelas (`clientes`, `produtos`, `pedidos`, `itens_pedido`).
* `\di` — Lista os índices existentes.
* `EXPLAIN (ANALYZE, BUFFERS) SELECT ...;` — Mostra o plano de execução real, tempo gasto e páginas lidas da memória/disco.
* `\q` — Sai do `psql`.

Também é possível rodar consultas diretamente sem entrar no modo interativo:
```bash
docker compose exec -T banco psql -U loja -d loja -c "SELECT relname, seq_scan, idx_scan FROM pg_stat_user_tables;"
```

### 4.2. Conectar por DBeaver ou pgAdmin
A porta `5432` do container `banco` está exposta para a máquina local. Use os seguintes dados de conexão:
* **Host:** `localhost` (ou `127.0.0.1`)
* **Porta:** `5432`
* **Database / Banco:** `loja`
* **Username / Usuário:** `loja`
* **Password / Senha:** `loja` (ou em branco, pois o container usa `POSTGRES_HOST_AUTH_METHOD: trust` no ambiente local)

### 4.3. Como mudar parâmetros e criar índices de forma reprodutível
Alterações feitas manualmente via `psql` ou DBeaver se perdem quando o volume é recriado e não podem ser reproduzidas por outra pessoa. Para que toda mudança no banco seja versionada e repetível:
1. **Não edite** `banco/01-estrutura.sql` nem `banco/02-dados.sql`.
2. Crie arquivos numerados em ordem alfabética a partir de `03-` dentro da pasta `banco/` (como `banco/03-indices.sql`).
3. Sempre que adicionar ou alterar um arquivo em `banco/`, recrie o volume do banco do zero para que o PostgreSQL execute todos os scripts de `/docker-entrypoint-initdb.d/`:
   ```bash
   docker compose down -v
   docker compose up -d --build
   ```

---

## 5. Problemas comuns

| Problema / Sintoma | Causa | Como resolver |
| :--- | :--- | :--- |
| **`A API não respondeu em http://127.0.0.1:3000/saude`** | Os containers não estão rodando ou o PostgreSQL ainda está populando os 2,12 milhões de registros iniciais. | Rode `docker compose ps` e `docker compose logs -f banco`. Aguarde aparecer `database system is ready to accept connections` e suba com `docker compose up -d --build`. |
| **O script `medir.sh` para no aquecimento no Windows (Git Bash)** | O Git Bash converte automaticamente `/carga/teste-carga.js` para `C:/Program Files/Git/carga/...`, que não existe dentro do container do k6. | Execute `export MSYS_NO_PATHCONV=1` no terminal antes de rodar `scripts/medir.sh`. |
| **Caracteres `^[[200~` ou `~` aparecem ao colar comandos no Git Bash** | O modo *bracketed paste* do terminal foi desconfigurado após rodar um container interativo. | Digite `reset` no terminal e rode `bind "set enable-bracketed-paste off"`. |
| **Criei o arquivo `banco/03-indices.sql`, mas os índices não aparecem no banco** | Scripts em `/docker-entrypoint-initdb.d/` só rodam quando o volume de dados do PostgreSQL é criado pela primeira vez. | Rode `docker compose down -v` para remover o volume antigo e depois `docker compose up -d --build`. |
| **`arquivos alterados sem commit` aparece maior que `0` em `contexto.txt`** | O `scripts/medir.sh` foi executado antes de fazer `git commit` das mudanças de código ou banco. | Sempre faça `git add` e `git commit` da alteração de código/SQL **antes** de rodar a bateria do `scripts/medir.sh`. |
| **Erro de porta `3000` ou `5432` já alocada** | Outro processo ou instância local do PostgreSQL/Node está ocupando a porta na máquina hospedeira. | Pare o serviço local conflitante ou defina `PORTA_API` ao medir. |

---

## 6. Resumo dos Experimentos e Resultados Obtidos

Todas as decisões técnicas, planos `EXPLAIN ANALYZE` (`diagnostico/`) e resultados brutos (`resultados/`) estão detalhados em `experimentos.md`, `proximos-testes.md` e no artigo técnico em `artigo/artigo.md`.

### Tabela Geral de Evolução (Mediana de 3 Execuções)

| Experimento / Rótulo | Vazão (req/s) | Mediana Geral | p95 Geral | CPU Média Banco | CPU Média API | Memória Máx API | Contrato (42 rotas) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`00-linha-de-base`** | 4,41 | 691,1 ms | 15.274,7 ms | 98,2% | 8,3% | 45,2 MiB | Mantido |
| **`01-pool-conexoes`** | 5,50 (+24,7%) | 451,4 ms (-34,7%) | 12.662,6 ms (-17,1%) | 98,4% | 6,3% | 48,0 MiB | Mantido |
| **`02-indices`** | 18,54 (+320,4%) | 101,4 ms (-85,3%) | 6.416,5 ms (-58,0%) | 99,8% | 20,4% | 90,7 MiB | Mantido |
| **`03-join-pedidos-cliente`** | **19,37 (+339,2%)** | 95,4 ms (-86,2%) | **6.233,7 ms (-59,2%)** | 99,5% | 20,0% | 92,5 MiB | Mantido |
| **`04-sql-vendas-por-dia`** | 18,55 (+320,6%) | **92,5 ms (-86,6%)** | 6.965,5 ms (-54,4%) | 100,3% | **3,3%** | **25,9 MiB** | Mantido |

### Latência por Rota — Antes (`00-linha-de-base`) vs. Depois (`04-sql-vendas-por-dia`)

| Rota | Endpoint | Mediana Antes | Mediana Depois | p95 Antes | p95 Depois | Redução na Mediana |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| `cliente` | `GET /clientes/:id` | 292,4 ms | **3,8 ms** | 601,6 ms | **101,7 ms** | **-98,7%** |
| `cliente-por-email` | `GET /clientes?email=` | 519,8 ms | **5,5 ms** | 2.230,8 ms | **120,6 ms** | **-98,9%** |
| `pedidos-do-cliente` | `GET /clientes/:id/pedidos` | 11.913,6 ms | **92,9 ms** | 19.126,2 ms | **202,7 ms** | **-99,2%** |
| `busca-produtos` | `GET /produtos/busca?q=` | 394,3 ms | **96,7 ms** | 743,3 ms | **202,8 ms** | **-75,5%** |
| `pedidos-por-status` | `GET /pedidos?status=&pagina=` | 7.385,1 ms | **595,3 ms** | 13.661,5 ms | **12.578,5 ms** | **-91,9%** |
| `vendas-por-dia` | `GET /relatorios/vendas-por-dia` | 4.421,0 ms | **2.492,9 ms** | 7.518,2 ms | **6.126,3 ms** | **-43,6%** |
| `top-produtos` | `GET /relatorios/top-produtos` | 12.760,4 ms | **16.813,3 ms** | 18.817,2 ms | **32.301,8 ms** | *Ver `proximos-testes.md`* |