# Objetivo

Criar uma automação em **Python** que simula **inserções contínuas** em um banco **PostgreSQL** com **estrutura OLTP hospitalar realista**, focada em testes de **CDC (Change Data Capture)** via Debezium (fora do escopo). A automação deve ser **resiliente**, **observável** e **configurável**, com **transações**, **FKs válidas** e **logs úteis** para validar volumes e consistência.

---

# Escopo & Diretrizes para o Copilot

> **Interprete este arquivo como o plano-fonte da implementação.** Gere todos os arquivos, pastas e códigos conforme descrito abaixo, prontos para executar em um ambiente local com Docker ou PostgreSQL nativo.

## Stack

* Linguagem: **Python 3.11+**
* DB: **PostgreSQL 14+** (CDC-ready)
* Driver: **psycopg2-binary**
* Faker: **faker** (locale `pt_BR`)
* CLI: **typer**
* Logs: **logging** (console), arquivos rotacionados em `/logs`

## Pastas do Projeto

```
/config
  ├─ .env.example
  └─ settings.toml
/sql
  ├─ 01_schema.sql
  ├─ 02_indexes.sql
  ├─ 03_seed-lookups.sql
  └─ 99_drop_all.sql
/scripts
  ├─ db_init.py          # cria schema, índices, lookups, valida conexão
  ├─ stream.py           # laço principal de inserção contínua
  ├─ seed.py             # popular volume inicial (>= 1.000 por tabela)
  ├─ reset.py            # drop + recreate + seed
  ├─ cli.py              # CLI Typer (iniciar, resetar, intervalo, pausa)
  ├─ data_gen.py         # funções de geração (pacientes, médicos, etc.)
  └─ validators.py       # validações de integridade e domínios
/logs
  └─ (gerados em runtime: app.log, stream.log, errors.log)
requirements.txt
README.md
Makefile
```

> **Observação:** mantenha imports relativos, type hints, docstrings e PEP 8 (linhas ≤ 88 col). Use `ruff`/`black` opcionalmente.

---

# Configuração

## Arquivo `/config/.env.example`

```
PG_HOST=localhost
PG_PORT=5432
PG_USER=postgres
PG_PASSWORD=postgres
PG_DATABASE=hospital_oltp

# Inserção contínua
STREAM_INTERVAL_SECONDS=2
BATCH_SIZE=50
MAX_JITTER_MS=400

# Semeadura (seed)
SEED_PACIENTES=2000
SEED_MEDICOS=200
SEED_CONVENIOS=12
SEED_CONSULTAS=4000
SEED_EXAMES=3500
SEED_INTERNACOES=1200
SEED_PACIENTES_CONVENIOS=2500

LOG_LEVEL=INFO
```

> O `.env` real deve ser copiado a partir de `.env.example`.

## Arquivo `/config/settings.toml`

```toml
[db]
search_path = "public"
connect_timeout = 10
application_name = "oltp-simulator"

[stream]
interval_seconds = 2
batch_size = 50
max_jitter_ms = 400
fail_fast_on_critical = true

[seed]
validate_foreign_keys = true

[logging]
level = "INFO"
rotate_when = "midnight"
backup_count = 7
```

---

# Esquema SQL (CDC-friendly)

## `/sql/01_schema.sql`

* **Regras gerais**: todas as tabelas com `id BIGSERIAL PRIMARY KEY`, `created_at TIMESTAMP NOT NULL DEFAULT now()`, `updated_at TIMESTAMP NOT NULL DEFAULT now()`, gatilho `updated_at` em `UPDATE` (trigger e função padrão).
* **Domínios e checks** para status e formatos básicos (ex.: status de consulta, CRM, CNPJ, CPF apenas como `VARCHAR` com validação em app).
* **FKs** com `ON UPDATE CASCADE`, `ON DELETE RESTRICT` (evitar deleções acidentais em cadastros-mestre).

```sql
-- Função e trigger para atualizar updated_at
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END; $$ LANGUAGE plpgsql;

-- Tabela: pacientes
CREATE TABLE IF NOT EXISTS pacientes (
  id            BIGSERIAL PRIMARY KEY,
  nome          VARCHAR(150) NOT NULL,
  nascimento    DATE NOT NULL,
  cpf           VARCHAR(14) UNIQUE NOT NULL,
  telefone      VARCHAR(20),
  endereco      VARCHAR(200),
  data_cadastro TIMESTAMP NOT NULL DEFAULT now(),
  created_at    TIMESTAMP NOT NULL DEFAULT now(),
  updated_at    TIMESTAMP NOT NULL DEFAULT now()
);
CREATE TRIGGER pacientes_updated_at
BEFORE UPDATE ON pacientes
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Tabela: medicos
CREATE TABLE IF NOT EXISTS medicos (
  id            BIGSERIAL PRIMARY KEY,
  nome          VARCHAR(150) NOT NULL,
  crm           VARCHAR(20) UNIQUE NOT NULL,
  especialidade VARCHAR(80) NOT NULL,
  telefone      VARCHAR(20),
  created_at    TIMESTAMP NOT NULL DEFAULT now(),
  updated_at    TIMESTAMP NOT NULL DEFAULT now()
);
CREATE TRIGGER medicos_updated_at
BEFORE UPDATE ON medicos
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Tabela: convenios
CREATE TABLE IF NOT EXISTS convenios (
  id         BIGSERIAL PRIMARY KEY,
  nome       VARCHAR(120) NOT NULL,
  cnpj       VARCHAR(18) UNIQUE NOT NULL,
  tipo       VARCHAR(40) NOT NULL, -- ex.: publico, privado, empresarial
  cobertura  VARCHAR(120),
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
CREATE TRIGGER convenios_updated_at
BEFORE UPDATE ON convenios
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Tabela N:N: pacientes_convenios
CREATE TABLE IF NOT EXISTS pacientes_convenios (
  id           BIGSERIAL PRIMARY KEY,
  paciente_id  BIGINT NOT NULL REFERENCES pacientes(id)
               ON UPDATE CASCADE ON DELETE RESTRICT,
  convenio_id  BIGINT NOT NULL REFERENCES convenios(id)
               ON UPDATE CASCADE ON DELETE RESTRICT,
  numero_carteira VARCHAR(40),
  validade     DATE,
  created_at   TIMESTAMP NOT NULL DEFAULT now(),
  updated_at   TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE (paciente_id, convenio_id)
);
CREATE TRIGGER pacientes_convenios_updated_at
BEFORE UPDATE ON pacientes_convenios
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Tabela: consultas
CREATE TABLE IF NOT EXISTS consultas (
  id          BIGSERIAL PRIMARY KEY,
  paciente_id BIGINT NOT NULL REFERENCES pacientes(id)
              ON UPDATE CASCADE ON DELETE RESTRICT,
  medico_id   BIGINT NOT NULL REFERENCES medicos(id)
              ON UPDATE CASCADE ON DELETE RESTRICT,
  data        TIMESTAMP NOT NULL,
  motivo      VARCHAR(200) NOT NULL,
  status      VARCHAR(20) NOT NULL CHECK (status IN ('agendada','realizada','cancelada','faltou')),
  created_at  TIMESTAMP NOT NULL DEFAULT now(),
  updated_at  TIMESTAMP NOT NULL DEFAULT now()
);
CREATE TRIGGER consultas_updated_at
BEFORE UPDATE ON consultas
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Tabela: exames
CREATE TABLE IF NOT EXISTS exames (
  id           BIGSERIAL PRIMARY KEY,
  paciente_id  BIGINT NOT NULL REFERENCES pacientes(id)
               ON UPDATE CASCADE ON DELETE RESTRICT,
  tipo_exame   VARCHAR(100) NOT NULL,
  data         TIMESTAMP NOT NULL,
  resultado    VARCHAR(200),
  created_at   TIMESTAMP NOT NULL DEFAULT now(),
  updated_at   TIMESTAMP NOT NULL DEFAULT now()
);
CREATE TRIGGER exames_updated_at
BEFORE UPDATE ON exames
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Tabela: internacoes
CREATE TABLE IF NOT EXISTS internacoes (
  id           BIGSERIAL PRIMARY KEY,
  paciente_id  BIGINT NOT NULL REFERENCES pacientes(id)
               ON UPDATE CASCADE ON DELETE RESTRICT,
  data_entrada TIMESTAMP NOT NULL,
  data_saida   TIMESTAMP,
  motivo       VARCHAR(200) NOT NULL,
  quarto       VARCHAR(20),
  created_at   TIMESTAMP NOT NULL DEFAULT now(),
  updated_at   TIMESTAMP NOT NULL DEFAULT now(),
  CHECK (data_saida IS NULL OR data_saida >= data_entrada)
);
CREATE TRIGGER internacoes_updated_at
BEFORE UPDATE ON internacoes
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

## `/sql/02_indexes.sql`

```sql
-- Índices úteis para consultas e CDC (chaves naturais e FK targets)
CREATE INDEX IF NOT EXISTS idx_pacientes_cpf ON pacientes (cpf);
CREATE INDEX IF NOT EXISTS idx_medicos_crm ON medicos (crm);
CREATE INDEX IF NOT EXISTS idx_consultas_paciente ON consultas (paciente_id);
CREATE INDEX IF NOT EXISTS idx_consultas_medico ON consultas (medico_id);
CREATE INDEX IF NOT EXISTS idx_consultas_data ON consultas (data);
CREATE INDEX IF NOT EXISTS idx_exames_paciente ON exames (paciente_id);
CREATE INDEX IF NOT EXISTS idx_exames_data ON exames (data);
CREATE INDEX IF NOT EXISTS idx_internacoes_paciente ON internacoes (paciente_id);
CREATE INDEX IF NOT EXISTS idx_internacoes_datas ON internacoes (data_entrada, data_saida);
```

## `/sql/03_seed-lookups.sql`

```sql
-- Exemplos de convênios-base (serão ampliados pelo seed.py)
INSERT INTO convenios (nome, cnpj, tipo, cobertura)
VALUES
  ('SUS', '00.000.000/0001-00', 'publico', 'integral'),
  ('SaudePlus', '11.111.111/0001-11', 'privado', 'ambulatorial e hospitalar')
ON CONFLICT DO NOTHING;
```

## `/sql/99_drop_all.sql`

```sql
DO $$ DECLARE r RECORD; BEGIN
  FOR r IN (
    SELECT tablename FROM pg_tables WHERE schemaname = 'public'
  ) LOOP
    EXECUTE 'DROP TABLE IF EXISTS ' || quote_ident(r.tablename) || ' CASCADE';
  END LOOP;
END $$;
DROP FUNCTION IF EXISTS set_updated_at() CASCADE;
```

---

# Requisitos Funcionais da Automação

1. **Inserção contínua** com **intervalo configurável** e **jitter** aleatório para realismo.
2. **Transações** por **lotes** (batch): commit a cada `BATCH_SIZE` inserções por tabela.
3. **Rollback** em erro crítico (ex.: perda de conexão, falha de migração de schema), e **continuação** em erros leves (ex.: `IntegrityError` por duplicidade de CPF/CRM).
4. **Validação prévia**: antes de inserir registros dependentes (consultas, exames, internações), **garanta** que `paciente_id`, `medico_id` e `convenio_id` existam. Caso não, **recrie** ou **selecione** válidos.
5. **Logs**: imprimir no terminal e gravar em arquivo por ciclo: exemplo `Inseridos 50 registros em consultas (total=12.340)`. Logar exceções com contexto (payload mínimo) sem dados sensíveis.
6. **Volumes**: `seed.py` deve gerar **≥ 1.000** registros por tabela (usar variáveis do `.env`).
7. **Reset total**: `reset.py` executa: drop all → schema → indexes → seed lookups → seed volume.
8. **Parada graciosa**: capturar `SIGINT/SIGTERM` e finalizar o ciclo atual com `commit` ou `rollback` seguro.

---

# Geração de Dados (Faker) & Regras

* Locale `pt_BR` para nomes, CPFs (com máscara), telefones e endereços.
* **CPF/CRM/CNPJ**: gerar formatos realistas; unicidade garantida em aplicação (set em memória) + índice UNIQUE no DB.
* **Status de consulta**: `agendada`, `realizada`, `cancelada`, `faltou` (distribuição ponderada, ex.: 55/35/5/5).
* **Tipos de exame**: hemograma, raio-x, tomografia, ultrassom, PCR, ECG, etc.
* **Internações**: tempo médio 3–10 dias; 20–30% sem `data_saida` (internação em andamento).
* **Convenios**: `tipo`: `publico|privado|empresarial`; `cobertura`: strings simples.
* **Relacionamentos**: cada paciente pode ter 0–3 convênios; consultas/exames/internações referenciam pacientes existentes.
* **Coerência temporal**: `data` de consultas/exames/internações ≥ `data_cadastro` do paciente; `data_saida >= data_entrada`.

---

# Scripts Python (especificação)

## `requirements.txt`

```
psycopg2-binary
python-dotenv
typer
faker
pydantic
```

## Convenções de Código

* Use `psycopg2.connect(...)` com `autocommit=False` e `isolation_level=READ COMMITTED`.
* Use **cursors contextuais** (`with conn, conn.cursor() as cur:`).
* No `stream.py`, implemente **retries exponenciais** para reconexão.
* Estruture logs com `logging.Logger` nomeado (`oltp.simulator`).
* Ao capturar `IntegrityError`, logue **campos-chave** e continue o loop.

## `scripts/db_init.py`

* Lê `.env` e `settings.toml`.
* Valida conexão (SELECT 1).
* Executa em ordem: `99_drop_all.sql` (se reset), `01_schema.sql`, `02_indexes.sql`, `03_seed-lookups.sql`.

## `scripts/seed.py`

* Cria **médicos**, **pacientes**, **convênios** adicionais e **pacientes_convenios**.
* Em seguida, gera **consultas**, **exames** e **internações** coerentes.
* **Batch** com `BATCH_SIZE` por tabela; **commit** por batch.
* Exibe contadores finais por tabela no console e grava em `/logs/app.log`.

## `scripts/data_gen.py`

* Funções puras para criar dicionários prontos para `INSERT`.
* Geração de identificadores (CPF/CRM/CNPJ) livres de duplicidade (set in-memory + consulta leve ao DB quando necessário).
* Utilitários de datas (ex.: janela de 2 anos para eventos, coerência com `data_cadastro`).

## `scripts/validators.py`

* Verificações de domínio (status permitido, datas válidas).
* Helpers para **verificar existência** de FK (paciente, médico, convênio) com cache LRU simples para performance.

## `scripts/stream.py`

* Loop infinito (até `Ctrl+C`), a cada `interval_seconds ± jitter`:

  1. Seleciona aleatoriamente entidades válidas (paciente, médico, convênio).
  2. Escolhe aleatoriamente **um tipo** de evento para inserir: `consulta | exame | internacao | novo_paciente` (ponderações configuráveis, se desejar).
  3. Insere **em transação**; em **falha leve**, loga e continua; em **erro crítico**, `rollback` e tenta reconectar.
  4. A cada batch, imprime: `consultas: +N (total=M)` etc.

## `scripts/reset.py`

* Executa `db_init.py` com `drop` e depois `seed.py`.
* Exibe resumo final de contagens por tabela.

## `scripts/cli.py` (Typer)

Comandos:

* `init-db` → cria schema e índices (sem drop)
* `seed` → popula volume inicial (≥ 1.000 por tabela)
* `stream` → inicia inserção contínua

  * Opções: `--interval 2`, `--batch-size 50`
* `reset` → drop + recreate + seed
* `counts` → exibe contagem por tabela (SELECT COUNT(*))
* `pause`/`resume` (opcional) → cria/remove um **arquivo-sentinela** `.paused` para o `stream.py` respeitar

### Exemplo de Uso

```
# 1) criar .env a partir do example
cp config/.env.example config/.env

# 2) preparar schema e sementes
python -m scripts.cli init-db
python -m scripts.cli seed

# 3) iniciar streaming contínuo (2s, batch 50)
python -m scripts.cli stream --interval 2 --batch-size 50

# 4) ver contagens
python -m scripts.cli counts

# 5) reset total
python -m scripts.cli reset
```

---

# Inserções: SQL parametrizado (psycopg2)

### Exemplo (consultas)

```python
cur.execute(
  """
  INSERT INTO consultas (paciente_id, medico_id, data, motivo, status)
  VALUES (%s, %s, %s, %s, %s)
  RETURNING id;
  """,
  (paciente_id, medico_id, ts, motivo, status)
)
```

> **Prática recomendada:** agrupe múltiplos `INSERTS` com `execute_values` do `psycopg2.extras` para alto throughput durante o **seed**. No **stream**, priorize granularidade 1–N pequena para realismo CDC.

---

# CDC & Debezium (pré-requisitos de modelagem)

* **PK obrigatório** em todas as tabelas (garantido por `BIGSERIAL` + UNIQUEs naturais em CPF/CRM/CNPJ).
* Campos `created_at`/`updated_at` e trigger para refletir `UPDATE` (útil para auditoria e CDC).
* Evitar `DELETE` massivos; preferir `status` ou **deleções raras** (simuladas quando necessário no `stream` com baixa probabilidade).
* **Não** habilitar CDC aqui; apenas **garantir modelagem** compatível (Debezium captura INSERT/UPDATE/DELETE via WAL).

---

# Observabilidade & Logs

* `logging` nível configurável via `.env`/`settings.toml`.
* **Formato sugerido**: `%(asctime)s %(levelname)s %(name)s [%(funcName)s] - %(message)s`
* **Exemplos de mensagens**:

  * `INFO consultas: inseridos +50 (total=12340) batch=50`
  * `WARNING pacientes: IntegrityError cpf=*** duplicado — registro ignorado`
  * `ERROR stream: perda de conexão — tentando reconectar em 2s (tentativa 3/5)`

---

# Makefile (atalhos)

```
.PHONY: install init seed stream counts reset fmt lint

install:
	python -m venv .venv && . .venv/bin/activate && pip install -r requirements.txt

init:
	python -m scripts.cli init-db

seed:
	python -m scripts.cli seed

stream:
	python -m scripts.cli stream --interval $${INTERVAL:-2} --batch-size $${BATCH:-50}

counts:
	python -m scripts.cli counts

reset:
	python -m scripts.cli reset

fmt:
	ruff check --fix . || true
	black .

lint:
	ruff check .
```

---

# Critérios de Aceite (Checklist)

* [ ] Pastas e arquivos criados conforme especificação
* [ ] Schema com PKs, FKs, índices e triggers `updated_at`
* [ ] `seed.py` gera ≥ 1.000 registros por tabela, sem quebras de integridade
* [ ] `stream.py` insere continuamente com transações, rollback em críticos, continuação em leves
* [ ] Logs mostram progresso por tabela e contagens totais
* [ ] `cli.py` expõe `init-db`, `seed`, `stream`, `reset`, `counts` (e `pause/resume` opcional)
* [ ] `reset.py` realiza drop + recreate + seed
* [ ] Código documentado, tipado e formatado

---

# Notas de Implementação

* Utilize **`psycopg2.sql`** para compor instruções seguras quando necessário.
* Empregue **`execute_values`** para inserts em lote no `seed.py`.
* Use **`random.choices`** com pesos para status/eventos.
* Garanta que *todas* as inserções dependentes consultem FKs válidas (com cache LRU para minimizar roundtrips).
* Mantenha **contadores globais** por tabela (em memória + SELECT COUNT(*) eventual) para logs fidedignos.
* Trate **timezone** (usar UTC no banco ou `TIMESTAMP WITHOUT TIME ZONE` consistente com aplicação).

---

# README (instruções resumidas)

1. Configure o `.env` em `/config`.
2. Crie venv e instale deps: `make install`.
3. Inicie o schema: `make init`.
4. Popule seed: `make seed`.
5. Rode o streaming: `make stream` (ajuste `INTERVAL`/`BATCH`).
6. Acompanhe logs no terminal e em `/logs`.
7. Para recriar tudo: `make reset`.

> **Resultado esperado:** Um banco OLTP coerente e mutável, com eventos contínuos (consultas, exames, internações e novos pacientes), fornecendo **material realista** para testes de **CDC/Debezium** e validações de consistência no pipeline de dados.
