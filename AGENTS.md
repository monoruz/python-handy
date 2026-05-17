## Cursor Cloud specific instructions

This repository is a **Python knowledge-base / cheat-sheet collection** — not a runnable application. It contains standalone markdown documentation files and independent Python utility scripts. There is no build system, no test suite, no dependency manifest (`requirements.txt`, `pyproject.toml`, etc.), and no service orchestration.

### Structure

- **Markdown files** (`README.md`, `Django.md`, `Testing.md`, `kafka.md`, `redis-cluster.md`, `concepts.md`): reference notes and code snippets.
- **Python scripts** (`logrotation.py`, `docker-reg-image-prune.py`, `loki-log.py`, `pyodbc_wrapper.py`, `redis-dump.py`, `retry-with-tenacity.py`): standalone utilities, each targeting a different external service or library.

### Running scripts

- `logrotation.py` uses only the Python standard library and can be run directly: `python3 logrotation.py` (runs in an infinite loop; Ctrl+C to stop).
- Other scripts require external services (Redis, Kafka, SQL Server, Grafana Loki, Docker Registry) and/or third-party packages (`redis`, `tenacity`, `flask`, `logging_loki`, `pyodbc`, `requests`). Install dependencies ad-hoc as needed since there is no shared requirements file.

### Known syntax issues in existing scripts

- `retry-with-tenacity.py` has an `IndentationError` on line 15.
- `loki-log.py` has a `SyntaxError` on line 29 (`if ip : :`).

### Lint / test / build

There is no linter configuration, no test framework, and no build step configured for this repository.
