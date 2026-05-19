# Quick Setup — Superset Backend (No Docker)

> **Status:** WIP. Instructions below are the current best guess; only items in the [Verification checklist](#verification-checklist) are confirmed working. Update this file as steps are validated.

Target: run the Superset Python/Flask backend locally against the default **SQLite** metadata DB, with enough setup to run the SQLite-friendly test suite.

## 1. Prerequisites

- **Python 3.10–3.12** (per `pyproject.toml`; Python 3.13 not supported yet)
- **System packages**:
  - Debian/Ubuntu:
    ```bash
    sudo apt-get update
    sudo apt-get install -y \
      python3-dev python3-venv build-essential pkg-config \
      default-libmysqlclient-dev libssl-dev libffi-dev libsasl2-dev libldap2-dev
    ```
    If your Python is a specific minor version (e.g. 3.12), prefer the matching dev package: `python3.12-dev`.
  - macOS (Homebrew): Python from `brew install python@3.12` already ships headers; no separate `-dev` package needed.
    ```bash
    brew install pkg-config mysql-client openssl
    ```

> **Why these packages are needed** — several Superset deps build C/C++ extensions during `pip install`:
> - `contourpy`, `pyinstrument`, `python-ldap` need **Python development headers** (`Python.h` → `python3-dev` / `python3.X-dev`). Missing this is the most common failure: `fatal error: Python.h: No such file or directory`.
> - `python-ldap` additionally needs `libsasl2-dev libldap2-dev libssl-dev`.
> - `mysqlclient` needs `default-libmysqlclient-dev` + `pkg-config` (or skip via the workaround below).

## 2. Create virtualenv and install

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements/development.txt
pip install -e .
```

**If you cannot install MySQL headers** (and don't need MySQL connectivity), skip `mysqlclient`:

```bash
pip install -r <(grep -v '^mysqlclient' requirements/development.txt)
pip install -e .
```

## 3. Initialize the metadata DB and create an admin user

```bash
superset db upgrade
superset fab create-admin       # interactive: set username/password/email
superset init
```

Default metadata DB: SQLite at `~/.superset/superset.db`. No DB server needed.

## 4. Start the backend

```bash
superset run -p 8088
```

Open <http://localhost:8088> and log in with the admin you created.

For autoreload during development:
```bash
superset run -p 8088 --with-threads --reload --debugger --debug
```

## 5. Run tests (SQLite-friendly subset)

Unit tests use in-memory SQLite and mocked sessions — no extra setup, matches what CI runs:

```bash
pytest tests/unit_tests tests/common --cache-clear
```

Optional environment variable used by CI: `SUPERSET_TESTENV=true`.

Integration tests (slower, some require Postgres/MySQL and will be skipped or fail) — only run if needed:
```bash
export SUPERSET_CONFIG=tests.integration_tests.superset_test_config
export SUPERSET_TESTENV=true
superset db upgrade
superset init
superset load-test-users
pytest tests/integration_tests
```

## Verification checklist

Mark items confirmed working on a fresh machine; leave unchecked until validated end-to-end.

- [ ] System packages install (`pkg-config`, MySQL dev headers, etc.)
- [ ] `python3 -m venv venv && source venv/bin/activate`
- [ ] `pip install -r requirements/development.txt` completes (or the `grep -v mysqlclient` fallback)
- [ ] `pip install -e .` completes
- [ ] `superset db upgrade` creates `~/.superset/superset.db`
- [ ] `superset fab create-admin` creates the admin user
- [ ] `superset init` completes
- [ ] `superset run -p 8088` serves the login page at <http://localhost:8088>
- [ ] Login as the admin user succeeds
- [ ] `pytest tests/unit_tests tests/common` passes

## Known issues / open questions

- **`Python.h: No such file or directory`** → install `python3-dev` (or `python3.X-dev` matching your interpreter). Confirmed cause of `contourpy`, `pyinstrument`, and `python-ldap` wheel build failures on a minimal Debian/Ubuntu image.
- **`mysqlclient` pkg-config failure** → install `default-libmysqlclient-dev pkg-config` (see Prerequisites) or skip it via `grep -v '^mysqlclient'`. Decide which we want as the default path before marking these instructions final.
- Confirm whether `pip install -e .` alone (without `requirements/development.txt`) is sufficient for `pytest tests/unit_tests tests/common`. If yes, the install step can be slimmed further.
- Confirm Python version actually in use on the target machine (failing logs show 3.12, which is supported).
