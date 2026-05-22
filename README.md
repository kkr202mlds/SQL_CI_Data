# SQL_CI_Data

### SQL Continuous Integration  Pipeline – Database Change Management with Automated Testing

---

#### Aim

- Treat database schema and reference data the same way as application code: version-controlled, automatically tested, and deployed via CI/CD.

---

#### Technology Used

| Area | Tool / Service |
| :--- | :--- |
| **Database** | [PostgreSQL](https://postgresql.org) *(easily replaceable)* |
| **Migration tool** | [Flyway](https://flywaydb.org) (or [Liquibase](https://liquibase.org)) |
| **Version control** | [Git](https://git-scm.com) + [GitHub](https://github.com/) |
| **CI platform** | [GitHub Actions](https://github.com) |
| **Test framework** | `pgTAP` (SQL unit tests), `Python/Pytest` (integration) |
| **Scripting** | Bash, Python |
| **Optional app layer** | [Python Flask](https://palletsprojects.com) / [Node.js](https://nodejs.org) *(to demonstrate full CI)* |


---

#### Project Structure
```
sql-ci-project/
├── .github/
│   └── workflows/
│       └── ci.yml                  # CI pipeline definition
├── migrations/                     # Flyway versioned SQL scripts
│   ├── V1__create_users_table.sql
│   ├── V2__add_email_to_users.sql
│   └── R__utility_functions.sql    # repeatable migration
├── test/
│   ├── sql/                        # pgTAP tests
│   │   └── users_tests.sql
│   └── integration/                # Python-based integration tests
│       └── test_database.py
├── scripts/
│   ├── start_db.sh                 # Spin up a temporary PostgreSQL
│   └── migrate.sh                  # Flyway migration runner
├── flyway.conf                     # Flyway configuration (without credentials)
├── requirements.txt                # Python deps (if used)
└── README.md
```
---

#### CI Pipeline Steps

The pipeline runs on every push/PR and does:

1. Spin up an ephemeral PostgreSQL instance (Docker container).
2. Checkout code.
3. Run Flyway migrations against the temporary database.
4. Execute SQL unit tests with pgTAP.
5. Execute integration tests (Python app connecting to DB, verifying schema).
6. Clean up (destroy container).

#### Detailed Implementation

Version-Controlled Migrations (Flyway)

File: migrations/V1__create_users_table.sql

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT now()
);
```

File: migrations/V2__add_email_to_users.sql

```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
```

##### Automated SQL Tests with pgTAP

Install pgTAP in the test container. Example test file test/sql/users_tests.sql:

```sql
BEGIN;
SELECT plan(2);

SELECT has_table('users');
SELECT col_is_pk('users', 'id');

SELECT * FROM finish();
ROLLBACK;
```

Run with: pg_prove -d $DATABASE_URL test/sql/*.sql

CI Workflow (GitHub Actions)

.github/workflows/ci.yml:

```yaml
name: SQL CI Pipeline

on: [push, pull_request]

jobs:
  database-ci:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: ci_user
          POSTGRES_PASSWORD: ci_pass
          POSTGRES_DB: ci_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Install Flyway
        run: |
          wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz | tar xvz
          echo "$(pwd)/flyway-9.22.0" >> $GITHUB_PATH

      - name: Run Flyway migrations
        run: |
          flyway -url=jdbc:postgresql://localhost:5432/ci_db \
                 -user=ci_user -password=ci_pass \
                 -locations=filesystem:migrations \
                 migrate

      - name: Install pgTAP and run SQL tests
        run: |
          sudo apt-get update && sudo apt-get install -y pgxnclient build-essential postgresql-client
          sudo pgxn install pgtap
          psql -h localhost -U ci_user -d ci_db -c "CREATE EXTENSION IF NOT EXISTS pgtap;"
          pg_prove -h localhost -U ci_user -d ci_db test/sql/*.sql

      # Optional: Python integration tests
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://ci_user:ci_pass@localhost:5432/ci_db
        run: pytest test/integration/ -v
```

#### Integration Test Example (Python)

test/integration/test_database.py:

```python
import psycopg2
import os

def test_users_table_exists():
    conn = psycopg2.connect(os.environ["DATABASE_URL"])
    cur = conn.cursor()
    cur.execute("SELECT EXISTS (SELECT FROM information_schema.tables WHERE table_name = 'users');")
    exists = cur.fetchone()[0]
    assert exists is True
    cur.close()
    conn.close()
```

---

#### What This Project Achieves

· Auditable database history: every schema change is a Git commit.
· Pre-merge validation: no broken SQL reaches main.
· Automated rollbacks: Flyway’s undo migrations can be tested in CI.
· Shift-left on data integrity: SQL tests catch logic errors before deployment.
· Dev parity: developers spin up the same DB locally via docker compose up + migration.

---

#### Extensions & Variations

· Replace Flyway with Liquibase (XML/YAML/JSON changelogs).
· Use GitLab CI instead of GitHub Actions – same pattern, different syntax.
· Add static analysis (e.g., sqlfluff linting) as a CI step.
· Deploy to a staging environment after CI passes (CD part).
· Include data seeding and verify reference data integrity.

---

## Testing Code

```bash
### Start test database
docker run --name pg-test -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=testdb -p 5432:5432 -d postgres:15

### Migrate
flyway -url=jdbc:postgresql://localhost:5432/testdb -user=test -password=test -locations=filesystem:migrations migrate

## Run SQL tests
pg_prove -h localhost -U test -d testdb test/sql/*.sql
```




