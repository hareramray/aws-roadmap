# 02 — Build, version, persist, and package a tiny application

[Prerequisite tutorials](README.md) · [All tutorials](../README.md)

**Mode:** local Python/SQLite/Git, with optional Docker. **Result:** a tested order store with parameterized SQL, a clean repository, and a container-persistence experiment. Use a new dedicated directory, not your roadmap repository, for the learner exercise.

## 1. Prepare the project

Create `orders-lab`, change into it, and initialize a Git repository. Create a `.gitignore` containing `.venv/`, `__pycache__/`, `*.db`, `.env`, and local secret/configuration paths used by your project. Do not add actual secrets to demonstrate ignoring them.

```powershell
git init
python -m venv .venv
.\.venv\Scripts\python.exe --version
git status --short
```

Calling the virtual environment's Python directly avoids changing PowerShell execution policy merely to activate a script. The exercise uses the standard library, so no external package is required.

## 2. Implement a transaction with an invariant

Save as `orders.py`:

```python
import sqlite3


def initialize(connection):
    connection.execute("PRAGMA foreign_keys = ON")
    connection.executescript("""
        CREATE TABLE IF NOT EXISTS orders (
            id TEXT PRIMARY KEY,
            customer TEXT NOT NULL
        );
        CREATE TABLE IF NOT EXISTS items (
            order_id TEXT REFERENCES orders(id),
            sku TEXT NOT NULL,
            quantity INTEGER NOT NULL CHECK(quantity > 0),
            price_cents INTEGER NOT NULL CHECK(price_cents >= 0)
        );
        CREATE INDEX IF NOT EXISTS orders_customer_idx ON orders(customer);
    """)


def create_order(connection, order_id, customer, quantity):
    with connection:
        connection.execute("INSERT INTO orders VALUES (?, ?)", (order_id, customer))
        connection.execute("INSERT INTO items VALUES (?, ?, ?, ?)",
                           (order_id, "BOOK", quantity, 1200))


def main():
    with sqlite3.connect(":memory:") as connection:
        initialize(connection)
        create_order(connection, "o-1", "blue", 2)
        try:
            create_order(connection, "o-2", "blue", 0)
        except sqlite3.IntegrityError:
            pass
        assert connection.execute("SELECT COUNT(*) FROM orders").fetchone()[0] == 1
        total = connection.execute("""
            SELECT o.customer, SUM(i.quantity * i.price_cents)
            FROM orders o JOIN items i ON i.order_id = o.id
            GROUP BY o.customer
        """).fetchall()
        assert total == [("blue", 2400)]
        print(total)
        print("PASS: invalid item rolled back the entire order")


if __name__ == "__main__":
    main()
```

Run `.\.venv\Scripts\python.exe orders.py`. Expect one customer aggregate of 2400 cents and the `PASS` line. The invalid item caused a rollback of both inserts; a partial order did not remain. Integer cents avoid floating-point arithmetic for this simple monetary example; real currency/tax rules require an explicit domain design.

## 3. Inspect the database model

Explain primary keys, foreign keys, `NOT NULL`, `CHECK`, indexes, joins, aggregation, and transaction boundaries using the script. Show why the query uses parameter binding for values rather than string interpolation. User data resembling SQL should remain data.

Run `EXPLAIN QUERY PLAN` for a customer lookup in a separate experiment and inspect use of the customer index. An index can improve selected reads while increasing write/storage work. SQLite's concurrency/storage behavior differs from RDS engines; this lesson teaches relational concepts, not RDS failover behavior.

Change the connection to a dedicated `orders.db` file to demonstrate persistence, but account for existing IDs on repeated runs. Either use a fresh test database or an explicit repeatable test fixture. Do not silently delete a database containing unknown work.

## 4. Version a reviewable change

```powershell
git add .gitignore orders.py
git diff --cached
git commit -m 'Add transactional order example'
git switch -c improve-validation
```

If Git identity is not configured, set an appropriate repository-local identity according to your preference; do not copy another person's name/email. Add explicit input checks to `create_order`, run the script, review `git diff`, commit, and merge the branch into your local main branch after checking its actual name.

Create a small intentional merge conflict in a copied exercise repository, inspect both changes, resolve the intended behavior, and rerun the example. Learn that resolving conflict markers is only syntax repair; behavior still needs checking. If a credential leaks in a real repository, revoke it before relying on history cleanup.

## 5. Package the deterministic example

If Docker is installed, create a `Dockerfile` using a current supported Python image you have verified. Pin the chosen image/tag/digest for a real reproducible project rather than relying indefinitely on a moving tag.

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY orders.py /app/orders.py
USER 10001:10001
CMD ["python", "orders.py"]
```

This concrete example only reads its code and uses an in-memory database, so the non-root user does not need to write `/app`. Add `.dockerignore` excluding `.git`, `.venv`, databases, secrets, and local artifacts. Build and run a local tag:

```powershell
docker build -t roadmap-orders:lab .
docker run --rm roadmap-orders:lab
```

Expect the same assertions/output. Inspect image contents and build context; a small final image can still contain secrets if they were copied into a layer. Do not put credentials into Dockerfile `ARG`/`ENV` values.

## 6. Test container state correctly

Use a separate disposable container and create a file in a writable location such as `/tmp`. Stop/start or restart that **same** container: the writable-layer file normally persists. Remove the container and create a new one from the image: that writable-layer file is gone. Repeat with a dedicated named volume and observe that the volume survives container replacement until explicitly removed.

Record container IDs, image digest, and volume name. Do not use a blanket Docker prune on a computer with unrelated projects. Remove only the lab container/image/volume after confirming ownership. A temporary memory-backed mount has different persistence behavior from a disk-backed writable layer.

## Verification and extension

Add tests for duplicate order IDs, missing customer values, negative quantity, and a SQL-looking customer string. Define which should fail and why. Build a small HTTP wrapper only after you can preserve these invariants; return a clear client error for invalid input and a request ID for diagnosis.

Explain build versus run, image versus container, persistent volume versus writable layer, source versus secret configuration, and unit versus integration tests. Continue with [distributed-system failure exercises](03-distributed-systems-security-and-testing.md).

References: [Python sqlite3](https://docs.python.org/3/library/sqlite3.html), [Git book](https://git-scm.com/book/en/v2), [Docker build guide](https://docs.docker.com/build/), [Docker storage](https://docs.docker.com/engine/storage/).
