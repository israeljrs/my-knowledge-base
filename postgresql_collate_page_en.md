# PostgreSQL COLLATE: a practical guide to locale, sorting, and comparison

This guide consolidates, organizes, and expands the points from the files `postgresql_collate_1.md` and `postgresql_collate_2.md` into a single, practical reference about collations in PostgreSQL: what they are, how to choose and configure them, how to resolve common errors, and best practices for production use.

## 1) Quick concepts

- **Encoding (e.g., UTF-8)**: character encoding mode. In PostgreSQL, it’s defined at the database level (and inherited from template databases). UTF-8 supports virtually all alphabets.
- **Collation (COLLATE)**: rules for string comparison and sorting. Defines how `ORDER BY`, `GROUP BY`, and comparisons handle case, accents/diacritics, etc.
- **OS locale vs PostgreSQL**: in glibc-based setups, PostgreSQL uses collations provided by the operating system (the collation name must exist on the OS). With ICU, you can use ICU collations (more consistent across platforms).

Important notes:
- There is no collation named “utf8” by itself. Use a locale collation with UTF-8, such as `pt_BR.UTF-8` (or `pt_BR.utf8`, depending on the OS), `en_US.UTF-8`, `C.UTF-8`/`C.utf8`, or an ICU collation such as `und-x-icu`.
- Language-aware collations (pt_BR, en_US) consume more CPU than `C`/`POSIX` (binary order), but provide linguistically correct ordering.

## 2) Where COLLATE applies

- **Database** (defaults for new objects): `LC_COLLATE` and `LC_CTYPE` at database creation time.
- **Column**: per-column `COLLATE` defines how values in that field are sorted/compared.
- **Expression/query**: `... ORDER BY column COLLATE "pt_BR.UTF-8"` applies the rule only for that `ORDER BY`.

## 3) Creating a database with UTF-8 and collation

Use `template0` to ensure independence from the default template’s settings:

```sql
CREATE DATABASE my_db
  WITH ENCODING 'UTF8'
       LC_COLLATE 'pt_BR.UTF-8'
       LC_CTYPE   'pt_BR.UTF-8'
       TEMPLATE template0;
```

The collation name can vary; adjust to `pt_BR.utf8` or `pt_BR.UTF-8` depending on your system (see section 6).

## 4) Setting COLLATE on columns and queries

Column with a specific collation:

```sql
CREATE TABLE users (
  name varchar(100) COLLATE "pt_BR.UTF-8"
);
```

Applying collation in a query:

```sql
SELECT name
FROM users
ORDER BY name COLLATE "pt_BR.UTF-8";
```

Practical difference example (data: 'Andre', 'Álvaro', 'Ana'):
- `pt_BR.UTF-8`: respects Portuguese rules (accents, diacritics, etc.).
- `C.UTF-8`/`C.utf8`: binary ordering (fast, but not language-aware).

## 5) Inspecting collations and encoding

In PostgreSQL:

```sql
SELECT collname, collcollate, collctype
FROM pg_collation
ORDER BY collname;

SHOW server_encoding;   -- should be UTF8
SHOW lc_collate;        -- database default collation
SHOW lc_ctype;          -- default character classification
```

On the operating system (Linux):

```bash
locale -a | grep -i pt_BR
```

Look for `pt_BR.UTF-8` or `pt_BR.utf8`. Use the exact name shown.

## 6) Common error: “collation ... does not exist”

Typical message:

```
ERROR: collation "pt_BR.UTF8" for encoding "UTF8" does not exist
```

How to fix:
1. **Confirm the correct name**: check `locale -a` and use exactly `pt_BR.UTF-8` or `pt_BR.utf8` (varies by distro).
2. **Install the locale on the OS** (Ubuntu/Debian):
   ```bash
   sudo apt-get install locales
   sudo locale-gen pt_BR.UTF-8
   sudo dpkg-reconfigure locales
   sudo systemctl restart postgresql
   ```
   On RHEL/CentOS:
   ```bash
   sudo yum install glibc-langpack-pt
   sudo localedef -i pt_BR -f UTF-8 pt_BR.UTF-8
   sudo systemctl restart postgresql
   ```
3. **Verify database encoding**:
   ```sql
   SHOW server_encoding; -- should be UTF8
   ```
   If it isn’t UTF8, create a new database as in section 3 and migrate the data.

## 7) Changing collation afterwards

### 7.1) On a column

```sql
ALTER TABLE names
ALTER COLUMN name TYPE varchar(50) COLLATE "pt_BR.UTF-8";
```

Recreate or reindex indexes that rely on this column:

```sql
REINDEX TABLE names;
```

### 7.2) On multiple text columns in the same table

Use a `DO` block to automate (adjust the table name):

```sql
DO $$
DECLARE r RECORD;
BEGIN
  FOR r IN (
    SELECT attname
    FROM pg_attribute
    WHERE attrelid = 'names'::regclass
      AND atttypid IN ('varchar'::regtype, 'text'::regtype)
      AND NOT attisdropped
  ) LOOP
    EXECUTE format(
      'ALTER TABLE names ALTER COLUMN %I TYPE %s COLLATE "pt_BR.UTF-8"',
      r.attname, pg_typeof(r.attname)::text
    );
  END LOOP;
END $$;
```

### 7.3) For the entire database

You cannot alter `LC_COLLATE`/`LC_CTYPE` of an existing database. Create a new database and migrate:

```sql
CREATE DATABASE new_db
  WITH ENCODING 'UTF8'
       LC_COLLATE 'pt_BR.UTF-8'
       LC_CTYPE   'pt_BR.UTF-8'
       TEMPLATE template0;
```

Then export and restore:

```bash
pg_dump -U user current_db > dump.sql
psql -U user -d new_db -f dump.sql
```

Recreate/reindex indexes as needed.

## 8) Indexes, performance, and consistency

- **Indexes depend on collation**: change collation → reindex to ensure consistency.
- **Functional indexes with COLLATE**: speed up frequent `ORDER BY ... COLLATE ...`.

```sql
CREATE INDEX idx_users_name_ptbr
  ON users ((name COLLATE "pt_BR.UTF-8"));
```

- **Performance**: `C.UTF-8`/`C.utf8` is faster (binary order), but not language-aware.
- **Consistency**: avoid mixing collations unnecessarily. Set clear defaults per database/column.

## 9) ICU vs glibc (when available)

- **ICU**: collations are consistent across systems and configurable. Example: `und-x-icu` (generic Unicode).
- **glibc (OS locales)**: depends on what the OS provides (`locale -a`).

If your installation has ICU enabled, prefer ICU for predictability across environments.

## 10) Quick behavior tests

```sql
-- Sample data
INSERT INTO names (name) VALUES ('Andre'), ('Álvaro'), ('Ana');

-- Ordering with Brazilian Portuguese
SELECT name FROM names ORDER BY name COLLATE "pt_BR.UTF-8";

-- Binary ordering
SELECT name FROM names ORDER BY name COLLATE "C.UTF-8"; -- or "C.utf8"
```

Compare the results to validate whether the ordering meets your use case.

## 11) Best practices

- **Deliberate choice**: use `pt_BR.UTF-8` for correct UX in Portuguese; use `C.UTF-8`/`C.utf8` for technical/bulk operations where binary order suffices.
- **Standardize names**: confirm the exact collation name on the OS with `locale -a`.
- **Plan migrations**: changing collation can impact indexes and queries; test beforehand and back up.
- **Avoid surprises**: apply `COLLATE` explicitly in critical queries if there’s a risk of environment variation.

---

Helpful references in your environment:
- `SELECT * FROM pg_collation;`
- `SHOW lc_collate; SHOW server_encoding;`
- `locale -a` (on the OS) to list available locales.

