# Development Guide

This guide is for developers who want to contribute to the project or set up a local development environment.

## Prerequisites

- **Python 3.11+** (check with `python --version`)
- **PostgreSQL 14+** (check with `psql --version`)
- **Git** for version control
- **Make** for convenient command execution

## Setup

### 1. Clone Repository

```bash
git clone https://github.com/yourusername/alimentador-bd.git
cd alimentador-bd
```

### 2. Create Virtual Environment

```bash
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

### 3. Install Dependencies

```bash
make install
# or
pip install -r requirements.txt
```

### 4. Configure Environment

```bash
cp config/.env.example config/.env
```

Edit `config/.env` with your PostgreSQL credentials:

```env
PG_HOST=localhost
PG_PORT=5432
PG_USER=postgres
PG_PASSWORD=yourpassword
PG_DATABASE=test_db
```

### 5. Initialize Database

```bash
make init
make seed
```

## Development Workflow

### Running Tests

```bash
# Validate connection
python scripts/test_connection.py

# Check table counts
make counts

# View logs
tail -f logs/app.log
```

### Starting Stream

For development/testing (short runs):

```bash
# 30 seconds
timeout 30 make stream

# Custom interval (1 second per operation)
STREAM_INTERVAL_SECONDS=1 timeout 60 make stream
```

### Code Quality

```bash
# Format code
make fmt

# Lint check
make lint

# Clean cache
make clean
```

## Project Structure

```
alimentador_bd/
├── config/                 # Configuration files
│   ├── .env.example       # Template for environment
│   └── settings.toml      # TOML configuration
├── scripts/               # Python modules
│   ├── __init__.py
│   ├── cli.py            # CLI entry point (Typer)
│   ├── db_init.py        # Database initialization
│   ├── stream.py         # Streaming engine
│   ├── seed.py           # Seeding logic
│   ├── data_gen.py       # Data generation (Faker)
│   ├── validators.py     # Validation & caching
│   ├── reset.py          # Reset orchestration
│   └── test_connection.py # Connection test utility
├── sql/                   # SQL scripts
│   ├── 01_schema.sql     # Table definitions
│   ├── 02_indexes.sql    # Index creation
│   ├── 03_seed-lookups.sql # Initial data
│   └── 99_drop_all.sql   # Cleanup script
├── logs/                  # Generated log files
├── README.md             # User documentation
├── CONTRIBUTING.md       # Contribution guidelines
├── ARCHITECTURE.md       # Technical architecture
├── DEVELOPMENT.md        # This file
├── CHANGELOG.md          # Version history
├── LICENSE               # MIT license
├── Makefile              # Build targets
├── requirements.txt      # Python dependencies
└── .gitignore            # Git exclusions
```

## Code Style

### Python Standards

We follow **PEP 8** with these guidelines:

- **Line length**: Maximum 88 characters (Black compatible)
- **Type hints**: Required for all functions
- **Docstrings**: Required for modules and public functions
- **Imports**: Organized (stdlib, third-party, local)
- **Naming**: snake_case for functions/variables, PascalCase for classes

### Example Function

```python
from typing import Optional
from psycopg2.extensions import connection as Connection

def generate_paciente(
    faker_instance: Faker,
    existing_cpfs: set[str],
) -> dict:
    """
    Generate a realistic patient record.

    Args:
        faker_instance: Configured Faker instance with pt_BR locale
        existing_cpfs: Set of already-used CPFs for uniqueness

    Returns:
        Dictionary with patient data ready for INSERT

    Raises:
        ValueError: If unable to generate unique CPF after 10 attempts
    """
    # Implementation...
    return {
        "nome": faker_instance.name(),
        "cpf": cpf,
        # ...
    }
```

### Comments

Use comments sparingly for **why**, not **what**:

```python
# Good: Explains reasoning
# Use exponential backoff to avoid overwhelming reconnection attempts
time.sleep(2 ** (attempt - 1))

# Avoid: Explains what code does (read from code itself)
# Increment i by 1
# i += 1
```

## Adding Features

### 1. New Operation Type

Example: Adding a DELETE operation (low probability).

#### Step 1: Add to `data_gen.py`

```python
def generate_deletion_target() -> int:
    """Select random record ID for deletion (realistic scenario)."""
    # Implementation...
    return record_id
```

#### Step 2: Add to `stream.py`

```python
def delete_consulta(conn: Connection, consulta_id: int) -> bool:
    """Delete a consultation (rare, realistic scenario)."""
    try:
        with conn.cursor() as cur:
            cur.execute(
                "DELETE FROM consultas WHERE id = %s RETURNING id;",
                (consulta_id,)
            )
            if cur.fetchone():
                conn.commit()
                return True
    except psycopg2.IntegrityError:
        conn.rollback()
        return False

# In main loop:
OPERATIONS = [
    (insert_paciente, 0.35),  # 35%
    (insert_consulta, 0.20),  # 20%
    (update_paciente, 0.25),  # 25%
    (delete_consulta, 0.05),  # 5% (rare)
    (update_consulta, 0.15),  # 15%
]
```

#### Step 3: Update weights in docs

Update `ARCHITECTURE.md` and `README.md` to reflect new operation distribution.

### 2. New Logging Level

Edit `config/settings.toml`:

```toml
[logging]
level = "DEBUG"  # Add verbose output
rotate_when = "midnight"
backup_count = 7
```

### 3. New Validator

Add to `scripts/validators.py`:

```python
@functools.lru_cache(maxsize=512)
def get_random_exame_type() -> str:
    """Cache exam types from database."""
    # Implementation...
```

## Testing

### Unit Testing (Future)

Prepare for pytest integration:

```bash
pip install pytest pytest-cov
pytest --cov=scripts tests/
```

### Integration Testing

Manual testing workflow:

```bash
# 1. Reset to clean state
make reset

# 2. Get initial counts
make counts
# Expected: ~13k records

# 3. Run stream for 60 seconds
timeout 60 make stream

# 4. Verify growth
make counts
# Expected: ~60-70 new records (1 per 2 seconds)

# 5. Check log file
tail -20 logs/app.log
```

### Data Integrity Testing

```sql
-- Ensure all FKs are valid
SELECT COUNT(*) FROM consultas 
WHERE paciente_id NOT IN (SELECT id FROM pacientes);
-- Expected: 0

-- Ensure all timestamps are sensible
SELECT COUNT(*) FROM consultas 
WHERE created_at > now();
-- Expected: 0

-- Verify unique constraints
SELECT COUNT(*) FROM pacientes
GROUP BY cpf HAVING COUNT(*) > 1;
-- Expected: No rows
```

## Debugging

### Enable Debug Logging

```python
# In cli.py, before running:
import logging
logging.getLogger("scripts").setLevel(logging.DEBUG)
```

### Inspect Database

```bash
# Connect directly
psql -U app -h localhost -d teste_pacientes

# Useful queries
\dt              # List tables
SELECT COUNT(*) FROM pacientes;
SELECT * FROM consultas ORDER BY created_at DESC LIMIT 5;
```

### Check Connection Issues

```bash
# Run connection test
python scripts/test_connection.py

# Verify network
ping 10.42.88.67
nc -zv 10.42.88.67 5432
```

## Common Issues

### Issue: "psycopg2.OperationalError: connection refused"

**Solution**: Check PostgreSQL is running and credentials are correct.

```bash
psql -U app -h 10.42.88.67 -d teste_pacientes -c "SELECT 1"
```

### Issue: "duplicate key value violates unique constraint"

**Solution**: This is handled gracefully by the stream. Check logs for CPF/CNPJ duplicates.

```bash
grep "IntegrityError" logs/app.log
```

### Issue: Stream not starting

**Solution**: Ensure database is initialized.

```bash
make init
make seed
make counts
```

## Performance Profiling

### Time a seeding operation

```bash
time make seed
```

Expected: ~2 seconds for 13k records.

### Monitor database during streaming

```bash
# Terminal 1
STREAM_INTERVAL_SECONDS=0 make stream

# Terminal 2
watch -n 1 'make counts'
```

Monitor growth rate.

## Contributing Changes

1. **Create feature branch**:
   ```bash
   git checkout -b feature/new-operation
   ```

2. **Make changes** following code style guidelines

3. **Test thoroughly**:
   ```bash
   make lint
   make test  # (when tests are added)
   ```

4. **Commit with clear message**:
   ```bash
   git commit -m "feat: add DELETE operations with 5% probability"
   ```

5. **Push and create Pull Request**:
   ```bash
   git push origin feature/new-operation
   ```

## Release Process

### Prepare Release

1. Update `CHANGELOG.md`
2. Update version in code (if applicable)
3. Test thoroughly
4. Create release branch: `git checkout -b release/v1.1.0`

### Tag Release

```bash
git tag -a v1.1.0 -m "Release v1.1.0: ADD description"
git push origin v1.1.0
```

### Publish

```bash
# (When publishing to PyPI, add setup.py and CI/CD)
python -m build
twine upload dist/*
```

## Resources

- [Python PEP 8](https://www.python.org/dev/peps/pep-0008/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [psycopg2 Docs](https://www.psycopg.org/psycopg2/docs/)
- [Faker Documentation](https://faker.readthedocs.io/)
- [Typer Documentation](https://typer.tiangolo.com/)

## Questions?

Check existing issues, review `README.md` troubleshooting, or open a new discussion.
