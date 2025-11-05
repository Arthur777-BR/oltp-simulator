# OLTP Hospital Simulator

Simulador realista de banco de dados hospitalar com inserções e atualizações contínuas para testes de CDC (Change Data Capture) via Debezium.

## Overview

Este projeto implementa um simulador de eventos hospitalares (OLTP) que insere e atualiza dados continuamente em um PostgreSQL, gerando um fluxo realista de mudanças para validação de pipelines de captura de dados.

### Características

- **7 Tabelas OLTP**: pacientes, médicos, convênios, consultas, exames, internações
- **Operações Realistas**: 70% INSERT, 30% UPDATE (sem DELETE)
- **CDC-Ready**: Schema otimizado com triggers, índices e chaves naturais
- **Resiliente**: Reconexão automática com backoff exponencial
- **Observável**: Logs detalhados por operação
- **Configurável**: Variáveis de ambiente para customização

## Pré-requisitos

- Python 3.11+
- PostgreSQL 14+
- pip

## Quick Start

### 1. Clonar e Configurar

```bash
git clone <seu-repo>
cd alimentador_bd
cp config/.env.example config/.env
```

### 2. Instalar Dependências

```bash
make install
```

### 3. Inicializar Banco

```bash
make init    # Criar schema
make seed    # Popular dados iniciais (~13k registros)
```

### 4. Iniciar Streaming

```bash
# Indefinidamente
make stream

# 1 minuto
timeout 60 make stream

# 5 minutos
timeout 300 make stream
```

## Configuração

Edite `config/.env`:

```env
PG_HOST=localhost
PG_PORT=5432
PG_USER=postgres
PG_PASSWORD=postgres
PG_DATABASE=teste_pacientes

STREAM_INTERVAL_SECONDS=2
BATCH_SIZE=50
MAX_JITTER_MS=400

SEED_PACIENTES=2000
SEED_MEDICOS=200
SEED_CONVENIOS=12
SEED_CONSULTAS=4000
SEED_EXAMES=3500
SEED_INTERNACOES=1200
SEED_PACIENTES_CONVENIOS=2500
```

## Comandos Disponíveis

```bash
make init       # Inicializa schema
make seed       # Popula dados iniciais
make stream     # Inicia streaming contínuo
make reset      # Drop + recreate + seed (cuidado!)
make counts     # Exibe contagens por tabela
make fmt        # Formata código
make lint       # Verifica código
make clean      # Limpa cache e venv
```

## Schema

### Tabelas Principais

- **pacientes** (2000 seed): CPF, nome, telefone, endereço
- **medicos** (200 seed): CRM, especialidade
- **convenios** (12 seed): CNPJ, tipo, cobertura
- **consultas** (4000 seed): paciente, médico, status
- **exames** (3500 seed): paciente, tipo, resultado
- **internacoes** (1200 seed): paciente, entrada/saída
- **pacientes_convenios** (2500 seed): relacionamento N:N

### Características

- PK: BIGSERIAL em todas as tabelas
- UKs: CPF, CRM, CNPJ (chaves naturais)
- FKs: ON UPDATE CASCADE, ON DELETE RESTRICT
- Triggers: `updated_at` automático em UPDATE
- Índices: Estratégicos em CPF, CRM, FK targets e datas

## Operações de Stream

### INSERT (70%)

- `insert_paciente`: Novo paciente com Faker pt_BR
- `insert_consulta`: Nova consulta com status válido
- `insert_exame`: Novo exame
- `insert_internacao`: Nova internação

### UPDATE (30%)

- `update_paciente`: Telefone ou endereço
- `update_consulta`: Status (agendada → realizada/cancelada/faltou)
- `update_exame`: Resultado (Normal/Alterado/Positivo/Negativo/Pendente)
- `update_internacao`: data_saida (alta)

## Logs

```
[    1] INSERT     consulta | INSERT:    1 | UPDATE:    0
[    2] INSERT     consulta | INSERT:    2 | UPDATE:    0
[    3] UPDATE     paciente | INSERT:    2 | UPDATE:    1
```

Formato: `[ciclo] TIPO tabela | INSERT total | UPDATE total`

## Troubleshooting

### "Banco não encontrado"

```bash
# Criar banco manualmente
psql -U postgres -c "CREATE DATABASE teste_pacientes"
```

### "Connection refused"

Verificar credenciais em `.env` e status do PostgreSQL:

```bash
pg_isready -h localhost -p 5432
```

### Limpar Cache

```bash
find . -type d -name __pycache__ -exec rm -rf {} +
rm -rf .venv logs/*.log
```

## Estrutura

```
.
├── config/
│   ├── .env                 # Credenciais (git-ignored)
│   ├── .env.example         # Template
│   └── settings.toml        # Configurações
├── sql/
│   ├── 01_schema.sql        # Schema principal
│   ├── 02_indexes.sql       # Índices
│   ├── 03_seed-lookups.sql  # Dados iniciais
│   └── 99_drop_all.sql      # Limpeza
├── scripts/
│   ├── cli.py               # CLI Typer
│   ├── db_init.py           # Inicialização
│   ├── seed.py              # Semeadura
│   ├── stream.py            # Streaming contínuo
│   ├── data_gen.py          # Geradores Faker
│   ├── validators.py        # Validações
│   └── reset.py             # Reset total
├── logs/                    # Logs em runtime
├── requirements.txt         # Dependências
├── Makefile                 # Atalhos
└── README.md               # Este arquivo
```

## Para Debezium/CDC

Este simulador é otimizado para captura via Debezium:

1. Habilitar logical replication no PostgreSQL
2. Criar publication em `teste_pacientes`
3. Configurar Debezium PostgreSQL Connector com:
   - `database.server.name`: `alimentador_bd`
   - `database.dbname`: `teste_pacientes`
   - `table.include.list`: `public.*`

Exemplo:

```json
{
  "name": "alimentador-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": 5432,
    "database.user": "postgres",
    "database.password": "postgres",
    "database.dbname": "teste_pacientes",
    "database.server.name": "alimentador_bd",
    "plugin.name": "pgoutput"
  }
}
```

## Licença

MIT

## Autor

Developed for OLTP testing and CDC validation
