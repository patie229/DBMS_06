# DBMS_06 – PostgreSQL in Practice: DDL, DML, and Bulk Import

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_06>  
**Prerequisites:** DBMS_01, DBMS_02, DBMS_03, DBMS_04, DBMS_05, Lecture 06  
**Duration:** 90 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Connect to a remote Linux server via **SSH** and work confidently in a
  terminal-only environment
- Verify that a running PostgreSQL instance is available and understand the
  role of the `postgres` system user
- Create a **database user** and a **database** using standard SQL commands
  from the `psql` client
- Re-implement a known relational schema in PostgreSQL by typing **DDL
  statements** interactively in `psql`
- Insert rows **one at a time** using `INSERT` and observe immediate feedback
  from the database engine
- Prepare a **CSV file by hand** and load its contents into a table using
  the standard SQL `COPY` statement
- Formulate and run **three analytical queries** directly in the `psql` shell
- Create and populate a second, independent database entirely from a
  **`.sql` script**

**After completing this exercise you should be able to answer the following questions independently:**

- What is the difference between the `postgres` operating-system user and a
  PostgreSQL database role?
- Why must foreign keys be defined in dependency order when creating tables?
- What are the advantages of `COPY FROM` over individual `INSERT` statements
  for large data volumes?
- What does `\q`, `\dt`, `\d <table>`, and `\c <database>` do inside `psql`?

---

## 0 – Connect to the Server

You can complete this exercise either on the **THGA lecture server** or on
**your own machine** running a Debian-based Linux distribution (Ubuntu, Debian,
WSL2, etc.).

### Option A – Lecture Server (SSH)

```bash
ssh <your-username>@<server-ip>
```

> You should see a shell prompt on the remote machine.
> If you are asked about the host fingerprint, type `yes` and press Enter.

> **Screenshot 1:** Take a screenshot of your terminal showing a successful
> SSH login (the welcome banner and your shell prompt) and insert it here.
>
> `[insert screenshot]`

### Option B – Your Own Machine

Open a terminal. All subsequent commands run locally — skip the `ssh` step.

---

## 1 – Verify PostgreSQL is Available

### Option A – Lecture Server

PostgreSQL is already installed and running on the lecture server. Just
confirm that the service is reachable and check the client version:

```bash
pg_isready
psql --version
```

> `pg_isready` should print something like
> `/var/run/postgresql:5432 - accepting connections`.

### Option B – Your Own Machine

If you are working on your own Debian-based system, install PostgreSQL first:

```bash
sudo apt-get update
sudo apt-get install -y postgresql postgresql-client
```

Then verify:

```bash
pg_isready
psql --version
```

> **Screenshot 2:** Take a screenshot showing the output of both commands.
>
> `[insert screenshot]`
> <img width="682" height="483" alt="Capture d’écran 2026-06-25 à 13 13 13" src="https://github.com/user-attachments/assets/b19d70e9-dba2-4fe7-ba5c-14fe6fbd2619" />



---

## 2 – Create a Database User and a Database

PostgreSQL uses the concept of **roles** for access control. After installation,
only the `postgres` superuser role exists. You will create a dedicated role for
this exercise and assign a new database to it.

Switch to the `postgres` system user to open a superuser session:

```bash
sudo -u postgres psql
```

You should now see the `psql` prompt:

```
psql (16.x)
Type "help" for help.

postgres=#
```

Create your role (replace `<your-username>` with your actual Unix login name
so that you can connect without specifying `-U` later):

```sql
CREATE ROLE <your-username> WITH LOGIN PASSWORD '<choose-a-password>';
```

Create the database and assign ownership to your new role:

```sql
CREATE DATABASE bibliothek_<your-username> OWNER <your-username>;
```

Verify:

```sql
SELECT rolname FROM pg_roles WHERE rolname = '<your-username>';
SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner
FROM   pg_database
WHERE  datname = 'bibliothek_<your-username>';
```

Exit the superuser session:

```sql
\q
```

> **Screenshot 3:** Take a screenshot showing the `CREATE ROLE`, `CREATE DATABASE`,
> and both `SELECT` results inside the `postgres=#` session.
>
> `[insert screenshot]`
> <img width="682" height="483" alt="Capture d’écran 2026-06-25 à 13 21 32" src="https://github.com/user-attachments/assets/6bbe84af-209f-47c4-a50c-da4edd146fca" />


---

## 3 – Connect as Your Own User

```bash
psql -U <your-username> -d bibliothek_<your-username>
```

You should see:

```
psql (16.x)
Type "help" for help.

bibliothek=>
```

> From this point on, every `psql` session in this exercise connects with
> `psql -U <your-username> -d bibliothek_<your-username>` unless stated otherwise.

---

## 4 – Create the Schema

You will now recreate the **municipal library** schema from DBMS_05 in
PostgreSQL. Type each statement individually and press Enter to execute it.
Do not paste all statements at once.

The schema consists of four tables:

| Table      | Description                                      |
|------------|--------------------------------------------------|
| `buch`     | Books, identified by ISBN                        |
| `exemplar` | Physical copies of a book                        |
| `mitglied` | Library members                                  |
| `ausleihe` | Lending transactions (copy ↔ member)             |

Type and execute the following statements one by one:

```sql
CREATE TABLE buch (
    isbn              TEXT           PRIMARY KEY,
    titel             TEXT           NOT NULL,
    erscheinungsjahr  INTEGER        NOT NULL,
    verlag            TEXT           NOT NULL,
    tagesgebuehr      NUMERIC(6,2)   NOT NULL CHECK (tagesgebuehr > 0)
);
```

```sql
CREATE TABLE exemplar (
    exemplar_id  INTEGER  PRIMARY KEY,
    isbn         TEXT     NOT NULL,
    standort     TEXT     NOT NULL,
    FOREIGN KEY (isbn) REFERENCES buch(isbn)
        ON DELETE RESTRICT ON UPDATE CASCADE
);
```

```sql
CREATE TABLE mitglied (
    mitglied_id     INTEGER      PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    nachname        TEXT         NOT NULL,
    vorname         TEXT         NOT NULL,
    geburtsdatum    DATE         NOT NULL,
    email           TEXT         NOT NULL UNIQUE,
    beitritt_datum  DATE         NOT NULL DEFAULT CURRENT_DATE
);
```

```sql
CREATE TABLE ausleihe (
    ausleihe_id      INTEGER  PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    exemplar_id      INTEGER  NOT NULL,
    mitglied_id      INTEGER  NOT NULL,
    ausleihe_datum   DATE     NOT NULL,
    rueckgabe_datum  DATE,
    FOREIGN KEY (exemplar_id) REFERENCES exemplar(exemplar_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (mitglied_id) REFERENCES mitglied(mitglied_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CHECK (rueckgabe_datum IS NULL OR rueckgabe_datum >= ausleihe_datum)
);
```

Verify that all four tables were created:

```sql
\dt
```

Inspect the structure of one table:

```sql
\d ausleihe
```

> **Screenshot 4:** Take a screenshot showing the output of `\dt` and
> `\d ausleihe`.
>
> `[insert screenshot]`
> <img width="1440" height="900" alt="Capture d’écran 2026-06-25 à 13 25 23" src="https://github.com/user-attachments/assets/81d47ce3-1c44-41eb-82c4-6f06293cd179" />


### Questions for Section 4

**Question 4.1:** You had to create `buch` before `exemplar`, and `exemplar`
and `mitglied` before `ausleihe`. Why does this order matter? What error would
PostgreSQL report if you tried to create `ausleihe` first?

> **Answer:** A `FOREIGN KEY` constraint can only reference a table that
> already exists. The referenced table and its referenced column (the
> primary key) must be defined before any table that points to it. Since
> `exemplar` references `buch(isbn)`, `buch` must exist first; since
> `ausleihe` references both `exemplar(exemplar_id)` and
> `mitglied(mitglied_id)`, both of those tables must exist before
> `ausleihe` is created.
>
> If you tried to create `ausleihe` first, PostgreSQL would abort the
> statement with an error such as:
>
> ```
> ERROR:  relation "exemplar" does not exist
> ```
>
> The dependency order is therefore the reverse of the deletion order: you
> create tables from the independent (referenced) side toward the dependent
> (referencing) side.

**Question 4.2:** The `mitglied_id` and `ausleihe_id` columns use
`GENERATED ALWAYS AS IDENTITY`. What does this mean? What happens if you try to
supply a value explicitly with `INSERT INTO mitglied (mitglied_id, ...) VALUES (5, ...)`?

> **Answer:** `GENERATED ALWAYS AS IDENTITY` means PostgreSQL manages the
> column's values automatically via an internal sequence. Each new row
> receives the next value; the application is not expected to provide one.
> This is the SQL-standard replacement for the older `SERIAL` pseudo-type.
>
> If you try to supply a value explicitly:
>
> ```sql
> INSERT INTO mitglied (mitglied_id, nachname, ...) VALUES (5, 'X', ...);
> ```
>
> PostgreSQL rejects it with:
>
> ```
> ERROR:  cannot insert a non-DEFAULT value into column "mitglied_id"
> DETAIL: Column "mitglied_id" is an identity column defined as GENERATED ALWAYS.
> HINT:  Use OVERRIDING SYSTEM VALUE to override.
> ```
>
> To force an explicit value you must write
> `INSERT INTO mitglied OVERRIDING SYSTEM VALUE VALUES (5, ...)`. With
> `GENERATED BY DEFAULT AS IDENTITY` (the weaker variant) explicit values
> would be allowed without the override.

**Question 4.3:** `tagesgebuehr` is defined as `NUMERIC(6,2)` while a simpler
`REAL` would also hold decimal numbers. Give a concrete example of an arithmetic
result that would differ between the two types when calculating a lending fee.

> **Answer:** `REAL` is a binary floating-point type and cannot represent
> most decimal fractions exactly. A 30-day loan at €0.10/day should cost
> exactly €3.00, but:
>
> ```sql
> SELECT CAST(0.1 AS REAL) * 30;        -- 3.00000004...  (inexact)
> SELECT CAST(0.1 AS NUMERIC) * 30;     -- 3.0            (exact)
> ```
>
> Another classic example: `SELECT 0.1 + 0.2;` yields
> `0.30000000000000004` in floating point but exactly `0.3` in `NUMERIC`.
> Across thousands of fee calculations these rounding errors accumulate and
> cause cent-level discrepancies in the books, which is unacceptable for
> monetary values. `NUMERIC(6,2)` stores the value as exact decimal and
> guarantees correct arithmetic.

---

## 5 – Insert Rows One at a Time

Insert each of the following rows by typing the `INSERT` statement manually and
executing it individually. Watch the `INSERT 0 1` confirmation after each one.

**Books:**

```sql
INSERT INTO buch VALUES ('978-3-423-08733-2', 'Steppenwolf', 1927, 'dtv', 0.50);
```

```sql
INSERT INTO buch VALUES ('978-3-518-36893-4', 'Homo Faber', 1957, 'Suhrkamp', 0.50);
```

```sql
INSERT INTO buch VALUES ('978-3-257-20456-6', 'Der Vorleser', 1995, 'Diogenes', 0.75);
```

```sql
INSERT INTO buch VALUES ('978-3-596-18296-4', 'Das Parfum', 1985, 'Fischer', 0.75);
```

```sql
INSERT INTO buch VALUES ('978-3-423-13571-9', 'Die Verwandlung', 1915, 'dtv', 0.30);
```

**Copies:**

```sql
INSERT INTO exemplar VALUES (1, '978-3-423-08733-2', 'A-01-3');
```

```sql
INSERT INTO exemplar VALUES (2, '978-3-423-08733-2', 'A-01-4');
```

```sql
INSERT INTO exemplar VALUES (3, '978-3-518-36893-4', 'A-02-1');
```

```sql
INSERT INTO exemplar VALUES (4, '978-3-257-20456-6', 'B-01-7');
```

```sql
INSERT INTO exemplar VALUES (5, '978-3-596-18296-4', 'B-02-2');
```

```sql
INSERT INTO exemplar VALUES (6, '978-3-423-13571-9', 'A-03-1');
```

**Members** — omit `mitglied_id` (it is generated) and omit `beitritt_datum`
for the first two members to test the `DEFAULT`:

```sql
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email)
VALUES ('Berger', 'Jonas', '2001-04-12', 'jonas.berger@mail.de');
```

```sql
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email)
VALUES ('Hartmann', 'Lea', '1998-07-08', 'lea.hartmann@example.com');
```

```sql
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email, beitritt_datum)
VALUES ('Sommer', 'Klara', '1985-11-30', 'klara.sommer@web.de', '2019-03-15');
```

After all inserts, verify the row count:

```sql
SELECT COUNT(*) FROM buch;
SELECT COUNT(*) FROM exemplar;
SELECT COUNT(*) FROM mitglied;
```

> **Screenshot 5:** Take a screenshot showing the three `COUNT(*)` results.
>
> `[insert screenshot]`
> <img width="682" height="567" alt="Capture d’écran 2026-06-25 à 13 34 08" src="https://github.com/user-attachments/assets/1b0da970-cba1-4bf2-98d9-b2566b1ded9e" />


Exit `psql`:

```sql
\q
```

---

## 6 – Create a CSV File and Load It with COPY

You will now add the lending records (`ausleihe`) not by hand, but by preparing
a CSV file and loading it with the standard SQL `COPY` statement.

### Step 1 – Write the CSV File

Open a new file in the terminal:

```bash
vim ausleihe.csv
```

Enter Insert mode with `i` and type the following content exactly — one row
per line, fields separated by commas, no header row:

```
1,2026-05-01,2026-05-10
2,2026-05-05,
3,2026-05-12,
6,2026-04-20,2026-04-28
```

The columns correspond to: `exemplar_id`, `ausleihe_datum`, `rueckgabe_datum`.
An empty trailing field means `NULL`.

Save and exit: press `Esc`, then type `:wq` and press Enter.

Verify the file:

```bash
cat ausleihe.csv
```

> Note: `mitglied_id` is not in the CSV. You will specify the target columns
> in the `COPY` statement and use a fixed value via a subsequent `UPDATE`, or
> you can extend the CSV to include the member assignment. The approach below
> adds `mitglied_id` to the CSV as a fourth column (member IDs: 1, 2, 1, 3).
> Recreate the file accordingly:

```
1,1,2026-05-01,2026-05-10
3,2,2026-05-05,
4,1,2026-05-12,
6,3,2026-04-20,2026-04-28
```

Columns: `exemplar_id`, `mitglied_id`, `ausleihe_datum`, `rueckgabe_datum`.

### Step 2 – Load the CSV

Connect to the database:

```bash
psql -U <your-username> -d bibliothek
```

Load the file using `COPY`. The path must be absolute:

```sql
COPY ausleihe (exemplar_id, mitglied_id, ausleihe_datum, rueckgabe_datum)
FROM '/home/<your-username>/ausleihe.csv'
WITH (FORMAT csv, NULL '');
```

> The `NULL ''` option tells PostgreSQL to interpret an empty field as `NULL`.

Verify:

```sql
SELECT * FROM ausleihe;
```

> **Screenshot 6:** Take a screenshot showing the full output of `SELECT * FROM ausleihe`.
>
> `[insert screenshot]`
><img width="682" height="567" alt="Capture d’écran 2026-06-25 à 13 44 50" src="https://github.com/user-attachments/assets/80831e16-6f17-4fd0-8165-2b947b9cc36a" />


### Questions for Section 6

**Question 6.1:** `COPY FROM` requires an absolute path on the server's
filesystem. What is the difference between server-side `COPY` and a
client-side import? In which scenario would you need the client-side variant?

> **Answer:** Server-side `COPY ... FROM '/path'` is executed by the
> PostgreSQL **server process**, so the file must reside on the **server's**
> filesystem and the server's OS user (`postgres`) must have permission to
> read it. The client never sees the file.
>
> The client-side variant is the `psql` meta-command `\copy`:
>
> ```
> \copy ausleihe (exemplar_id, mitglied_id, ausleihe_datum, rueckgabe_datum) \
>       FROM 'ausleihe.csv' WITH (FORMAT csv, NULL '')
> ```
>
> `\copy` reads the file on the **client** machine and streams its contents
> to the server over the existing connection. You need it whenever:
> - the data file is on your local machine but the database runs on a remote
>   server you cannot copy files to, or
> - your database role does not have server-side superuser /
>   `pg_read_server_files` privileges (server-side `COPY FROM` a file
>   requires elevated rights, whereas `\copy` works with ordinary client
>   privileges).

**Question 6.2:** The `NULL ''` option maps empty CSV fields to `NULL`.
What would happen without this option if the `rueckgabe_datum` field is empty?

> **Answer:** Without `NULL ''`, an empty field for a `DATE` column would be
> passed to the date input function as an empty string `''`, which is not a
> valid date. PostgreSQL would abort the `COPY` with an error such as:
>
> ```
> ERROR:  invalid input syntax for type date: ""
> CONTEXT:  COPY ausleihe, line 2, column rueckgabe_datum: ""
> ```
>
> The explicit `NULL ''` option makes the intent unambiguous: an empty
> field is to be stored as SQL `NULL` (loan still open), not as an empty
> string. This is exactly what we want for the open loans (lines 2 and 3).

**Question 6.3:** `ausleihe_id` is `GENERATED ALWAYS AS IDENTITY` and was not
included in the CSV or the `COPY` column list. How does PostgreSQL handle the
missing value? What would happen if you tried to include `ausleihe_id` in the
`COPY` column list with explicit values?

> **Answer:** Because `ausleihe_id` is omitted from the `COPY` column list,
> PostgreSQL generates its value automatically from the identity sequence
> for every imported row — exactly as it would for an `INSERT` that omits
> the column. The four CSV rows therefore receive `ausleihe_id` 1, 2, 3, 4.
>
> If you included `ausleihe_id` in the `COPY` column list with explicit
> values, `COPY` would fail for the same reason a direct `INSERT` would:
>
> ```
> ERROR:  cannot insert a non-DEFAULT value into column "ausleihe_id"
> DETAIL: Column "ausleihe_id" is an identity column defined as GENERATED ALWAYS.
> ```
>
> You would have to redefine the column as `GENERATED BY DEFAULT AS IDENTITY`
> to permit supplying explicit values.

---

## 7 – Three Queries in the psql Shell

Type and execute each query individually in the `psql` shell. Activate
formatted output first:

```sql
\pset format aligned
\pset border 1
```

---

### Query 1 – Currently Open Loans

List all loans that have not yet been returned: show the member's full name
(last name, first name), the book title, and the number of days the book has
been borrowed so far (today minus `ausleihe_datum`).

```sql
SELECT m.nachname,
       m.vorname,
       b.titel,
       CURRENT_DATE - a.ausleihe_datum AS tage_ausgeliehen
FROM   ausleihe   a
JOIN   exemplar   e ON e.exemplar_id = a.exemplar_id
JOIN   buch       b ON b.isbn        = e.isbn
JOIN   mitglied   m ON m.mitglied_id = a.mitglied_id
WHERE  a.rueckgabe_datum IS NULL
ORDER BY tage_ausgeliehen DESC;
```

> **Result:** There are **two open loans** (the rows with an empty
> `rueckgabe_datum`: `exemplar_id` 3 and 4). One belongs to Klara Sommer
> (copy 3 = *Homo Faber*, borrowed 2026-05-05) and one to Jonas Berger
> (copy 4 = *Der Vorleser*, borrowed 2026-05-12). The member who has held a
> book the longest is **Klara Sommer**, since her loan started earliest
> (2026-05-05) and therefore has the largest `tage_ausgeliehen` value.

---

### Query 2 – Loans per Member

For each member, show their full name, the total number of loans, and the
number of loans that are still open. Sort descending by total loans.

```sql
SELECT m.nachname,
       m.vorname,
       COUNT(*)                                          AS ausleihen_gesamt,
       COUNT(*) FILTER (WHERE a.rueckgabe_datum IS NULL) AS noch_offen
FROM   mitglied m
JOIN   ausleihe a ON a.mitglied_id = m.mitglied_id
GROUP BY m.mitglied_id, m.nachname, m.vorname
ORDER BY ausleihen_gesamt DESC;
```

> **Result:** **Jonas Berger** has the most loans (2 total: copies 1 and 4,
> with 1 still open). Klara Sommer and Lea Hartmann each have 1 loan.
>
> **`FILTER (WHERE ...)` vs `CASE WHEN`:** `COUNT(*) FILTER (WHERE
> a.rueckgabe_datum IS NULL)` counts only the rows that satisfy the
> condition. It is the SQL-standard, more readable equivalent of
> `COUNT(CASE WHEN a.rueckgabe_datum IS NULL THEN 1 END)`. Both produce the
> same result — `COUNT` ignores `NULL`s, so the `CASE` returns `NULL` for
> non-matching rows and they are not counted — but `FILTER` states the
> intent directly and is easier to read.

---

### Query 3 – Books That Have Never Been Borrowed

Return the title and publisher of every book for which no lending record
exists anywhere in `ausleihe` (regardless of return date).

```sql
SELECT b.titel,
       b.verlag
FROM   buch b
WHERE  NOT EXISTS (
    SELECT 1
    FROM   exemplar e
    JOIN   ausleihe a ON a.exemplar_id = e.exemplar_id
    WHERE  e.isbn = b.isbn
);
```

> **Result:** Borrowed copies are 1, 3, 4, 6, which belong to *Steppenwolf*
> (copies 1, 2), *Homo Faber* (copy 3), *Der Vorleser* (copy 4), and *Die
> Verwandlung* (copy 6). The only book with **no** lending record for any of
> its copies is **Das Parfum** (Fischer) — copy 5 was never borrowed. So the
> query returns a single row: *Das Parfum*, Fischer.

> **Screenshot 7:** Take a screenshot showing the output of all three queries
> in sequence in the `psql` shell.
>
> `[insert screenshot]`

### Questions for Section 7

**Question 7.1:** Query 1 joins four tables. In what order must the joins be
performed to always produce a correct result, and does the join order affect
correctness or only performance?

> **Answer:** The join order does **not affect correctness** — only
> performance. Inner joins are commutative and associative, so any order of
> joining `ausleihe`, `exemplar`, `buch`, and `mitglied` produces the same
> result set (the join conditions on the foreign keys uniquely pair the
> rows).
>
> What the optimizer *does* care about is performance: it will reorder the
> joins to keep intermediate results small (e.g. apply the
> `WHERE a.rueckgabe_datum IS NULL` filter on `ausleihe` first, then join
> outward). In PostgreSQL the query planner chooses the physical join order
> automatically based on table statistics, regardless of the textual order
> in the `FROM` clause. The logical chain is
> `ausleihe → exemplar → buch` (to reach the title) and
> `ausleihe → mitglied` (to reach the member name).

**Question 7.2:** Query 2 groups by `m.mitglied_id` in addition to the name
columns. Why is grouping by the primary key necessary even though names appear
unique in the sample data?

> **Answer:** Two different members could share the same first and last name
> (e.g. two "Müller, Thomas"). Grouping only by name would merge their loans
> into a single group, producing a wrong total. Grouping by the primary key
> `mitglied_id` guarantees that each member forms exactly one group,
> regardless of name collisions.
>
> There is also a formal SQL reason: every non-aggregated column in the
> `SELECT` list must appear in the `GROUP BY` clause (or be functionally
> dependent on it). Since `mitglied_id` is the primary key, `nachname` and
> `vorname` are functionally dependent on it; PostgreSQL allows them in the
> `SELECT` once `mitglied_id` is in the `GROUP BY`. Grouping by the key is
> therefore both semantically correct and standard-compliant.

**Question 7.3:** Query 3 uses `NOT EXISTS` with a correlated subquery. Rewrite
the query using `EXCEPT` and verify that both variants return the same result.
Write your rewritten query here:

> **Answer:**
>
> ```sql
> SELECT titel, verlag FROM buch
> EXCEPT
> SELECT b.titel, b.verlag
> FROM   buch b
> JOIN   exemplar e ON e.isbn        = b.isbn
> JOIN   ausleihe a ON a.exemplar_id = e.exemplar_id;
> ```
>
> The first `SELECT` lists all books; the second lists all books that *have*
> been borrowed at least once. `EXCEPT` returns the set difference — the
> books that appear in the first set but not the second — which is exactly
> the books never borrowed. Both variants return the single row *Das Parfum*,
> Fischer.
>
> Note: `EXCEPT` removes duplicates and compares whole rows, so it works
> cleanly here because `(titel, verlag)` identifies each book uniquely in
> the sample data. `NOT EXISTS` is generally preferable when the table has
> `NULL`s in the compared columns or when additional correlated conditions
> are needed.

Exit `psql`:

```sql
\q
```

---

## 8 – A Second Database from a Script

You will now create a new, independent database for a different domain: a
small **cinema programme** that stores films, screenings, and seat reservations.

### Step 1 – Create the Script File

```bash
vim kino.sql
```

Enter the following content:

```sql
-- Create the database (run this outside a transaction, directly in psql)
-- The database is created manually; this script sets up the schema and data.

CREATE TABLE film (
    film_id     INTEGER      PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    titel       TEXT         NOT NULL,
    laenge_min  INTEGER      NOT NULL CHECK (laenge_min > 0),
    fsk         INTEGER      NOT NULL CHECK (fsk IN (0, 6, 12, 16, 18))
);

CREATE TABLE vorstellung (
    vorstellung_id  INTEGER   PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    film_id         INTEGER   NOT NULL,
    beginn          TIMESTAMP NOT NULL,
    saal            TEXT      NOT NULL,
    FOREIGN KEY (film_id) REFERENCES film(film_id)
        ON DELETE RESTRICT ON UPDATE CASCADE
);

CREATE TABLE reservierung (
    reservierung_id  INTEGER  PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    vorstellung_id   INTEGER  NOT NULL,
    sitzplatz        TEXT     NOT NULL,
    name_gast        TEXT     NOT NULL,
    FOREIGN KEY (vorstellung_id) REFERENCES vorstellung(vorstellung_id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    UNIQUE (vorstellung_id, sitzplatz)
);

-- Sample data
INSERT INTO film (titel, laenge_min, fsk) VALUES
    ('Metropolis',         153, 0),
    ('Das Boot',           149, 16),
    ('Lola rennt',          81, 12),
    ('Der Untergang',      156, 12),
    ('Goodbye Lenin!',     121, 6);

INSERT INTO vorstellung (film_id, beginn, saal) VALUES
    (1, '2026-07-01 20:00', 'Saal 1'),
    (2, '2026-07-01 18:00', 'Saal 2'),
    (3, '2026-07-02 21:00', 'Saal 1'),
    (4, '2026-07-02 19:30', 'Saal 3'),
    (5, '2026-07-03 17:00', 'Saal 2');

INSERT INTO reservierung (vorstellung_id, sitzplatz, name_gast) VALUES
    (1, 'A1', 'Mertens, Paul'),
    (1, 'A2', 'Mertens, Paul'),
    (1, 'B5', 'Fischer, Ruth'),
    (2, 'C3', 'Wagner, Erik'),
    (3, 'A1', 'Schulze, Lena'),
    (3, 'D7', 'Schulze, Lena'),
    (4, 'B2', 'Braun, Otto'),
    (5, 'A3', 'Klein, Marie');
```

Save and exit: `Esc`, `:wq`, Enter.

### Step 2 – Create the Database

```bash
sudo -u postgres psql -c "CREATE DATABASE kino OWNER <your-username>;"
```

### Step 3 – Run the Script

```bash
psql -U <your-username> -d kino -f kino.sql
```

> You should see a sequence of `CREATE TABLE` and `INSERT` confirmations.

> **Screenshot 8:** Take a screenshot showing the script execution output.
>
> `[insert screenshot]`
> <img width="682" height="567" alt="Capture d’écran 2026-06-25 à 14 08 47" src="https://github.com/user-attachments/assets/3f3f2797-5680-4f0c-a9fe-b73558c1ec0b" />


---

## 9 – Inspect the New Database

Connect to the `kino` database:

```bash
psql -U <your-username> -d kino
```

List all tables:

```sql
\dt
```

Inspect the `reservierung` table structure:

```sql
\d reservierung
```

Examine the data:

```sql
SELECT * FROM film;
```

```sql
SELECT v.vorstellung_id,
       f.titel,
       v.beginn,
       v.saal
FROM   vorstellung v
JOIN   film        f ON f.film_id = v.film_id
ORDER BY v.beginn;
```

```sql
SELECT f.titel,
       COUNT(r.reservierung_id) AS reservierungen
FROM   film         f
JOIN   vorstellung  v ON v.film_id         = f.film_id
LEFT JOIN reservierung r ON r.vorstellung_id = v.vorstellung_id
GROUP BY f.film_id, f.titel
ORDER BY reservierungen DESC;
```

> **Screenshot 9:** Take a screenshot showing the output of all three
> `SELECT` statements.
>
> `[insert screenshot]`
> <img width="682" height="931" alt="Capture d’écran 2026-06-25 à 14 13 36" src="https://github.com/user-attachments/assets/e69c3ee3-5b35-4deb-9c13-92c287405943" />


### Questions for Section 9

**Question 9.1:** The `reservierung` table has a `UNIQUE (vorstellung_id, sitzplatz)`
constraint. What does this prevent, and at which level is this constraint
enforced — application or database?

> **Answer:** The composite `UNIQUE (vorstellung_id, sitzplatz)` constraint
> prevents the **same seat from being reserved twice for the same
> screening** — i.e. double-booking seat A1 for screening 1. Two reservations
> may share the same seat label *only if* they belong to different
> screenings (seat A1 in screening 1 and seat A1 in screening 3 are
> independent).
>
> The constraint is enforced at the **database level**. No matter which
> application or script attempts the insert, PostgreSQL rejects a conflicting
> row with a `duplicate key value violates unique constraint` error. This is
> strictly stronger than an application-level check, which could be bypassed
> by a second application, a direct SQL session, or a race condition between
> two concurrent requests.

**Question 9.2:** The third query uses `LEFT JOIN` between `vorstellung` and
`reservierung`. What would be different about the result if you used `JOIN`
(inner join) instead? Which films would disappear from the result and why?

> **Answer:** With a `LEFT JOIN`, screenings that have **no** reservations
> are still kept, and their reservation count comes out as 0 (because
> `COUNT(r.reservierung_id)` counts only non-NULL ids). With an inner `JOIN`,
> any screening with no matching reservation row would be dropped entirely.
>
> In the sample data every screening (1–5) has at least one reservation, so
> the *set of films* would not change in this particular dataset. However,
> if a film had a screening with zero reservations, that screening would
> vanish under an inner join, and a film whose screenings were **all** empty
> would disappear from the result completely. The `LEFT JOIN` guarantees that
> every film/screening appears even with a count of 0 — which is exactly what
> a "reservations per film" report should show.

**Question 9.3:** `ON DELETE CASCADE` was chosen for `reservierung.vorstellung_id`,
but `ON DELETE RESTRICT` for `vorstellung.film_id`. Justify both choices in
terms of the domain.

> **Answer:**
> - **`reservierung.vorstellung_id` → `ON DELETE CASCADE`:** A reservation
>   only has meaning in the context of a specific screening. If a screening
>   is cancelled and removed, its seat reservations become meaningless and
>   should be deleted automatically along with it. Cascading the delete is
>   the correct domain behaviour — no orphaned reservations remain.
> - **`vorstellung.film_id` → `ON DELETE RESTRICT`:** A film may have several
>   scheduled screenings. Deleting a film while screenings still reference it
>   would leave those screenings (and their reservations) dangling, and could
>   erase part of the published programme by accident. `RESTRICT` forces the
>   operator to first cancel/remove the screenings (a deliberate act) before
>   the film can be deleted, protecting against accidental data loss.

Exit `psql`:

```sql
\q
```

---

## 10 – Reflection

**Question A – Server vs. embedded database:**  
SQLite (DBMS_05) and PostgreSQL (this exercise) are both relational databases,
but they operate very differently. Name two concrete differences you experienced
in this exercise — in terms of setup, access control, or SQL behaviour.

> **Answer:**
> 1. **Architecture & setup:** SQLite is *embedded* — the whole database is
>    a single file accessed directly by the client process, with no server
>    and no configuration. PostgreSQL is a *client–server* system: a
>    background server process listens on port 5432, and clients like `psql`
>    connect to it over a socket. This exercise required `pg_isready`,
>    starting from the `postgres` system user, and connecting with `psql -U`.
> 2. **Access control:** SQLite has no concept of users or roles — anyone who
>    can open the file can read and write it. PostgreSQL enforces
>    role-based access control: we had to `CREATE ROLE ... WITH LOGIN
>    PASSWORD` and assign database ownership. (A third valid difference:
>    identity columns — SQLite uses `INTEGER PRIMARY KEY` autoincrement,
>    PostgreSQL uses `GENERATED ALWAYS AS IDENTITY`; and PostgreSQL enforces
>    `FOREIGN KEY`s by default, whereas SQLite needs `PRAGMA foreign_keys = ON`.)

**Question B – COPY vs. INSERT:**  
You inserted the `buch` and `exemplar` rows one at a time, and the `ausleihe`
rows via `COPY`. For a real import of 50,000 rows, which approach would you
choose and why? What is the main operational cost of individual `INSERT`
statements at scale?

> **Answer:** For 50,000 rows I would use `COPY` (or `\copy`). `COPY` is
> designed for bulk loading: it parses the whole file in a single operation
> and writes rows in large batches, typically one to two orders of magnitude
> faster than individual `INSERT`s.
>
> The main operational cost of individual `INSERT` statements at scale is
> **per-statement overhead**: each `INSERT` is parsed, planned, and — under
> autocommit — wrapped in its own transaction with its own commit and
> disk-flush (WAL fsync). For 50,000 rows that means 50,000 round-trips and
> 50,000 commits. `COPY` performs the load within a single transaction and a
> single command, eliminating that repeated overhead. (If `INSERT`s must be
> used, wrapping them in one transaction and using multi-row `VALUES`
> recovers much of the speed, but `COPY` remains the standard tool.)

**Question C – Role model:**  
You created a dedicated role with `LOGIN` and a password. The `postgres`
superuser also exists. What is the security principle behind creating a
separate role instead of always connecting as `postgres`?

> **Answer:** The principle is **least privilege**: each user or application
> should have only the rights it actually needs, and no more. The `postgres`
> superuser can read, modify, and drop *anything* in the cluster, bypass all
> permission checks, and even alter system configuration. Connecting as
> `postgres` for everyday work means a mistake (a wrong `DROP`, a bad
> `UPDATE`) or a compromised credential has unlimited blast radius.
>
> A dedicated role that owns only its own database limits the damage: it can
> manage its own tables but cannot touch other databases or system catalogs.
> It also improves accountability (actions are attributable to a specific
> role) and makes it possible to revoke or change one user's access without
> affecting the superuser.

**Question D – Script-driven setup:**  
The `kino.sql` script creates the schema and inserts data in one run. What
is the advantage of this approach over typing the statements interactively?
Name one situation where an interactive approach is still preferable.

> **Answer:** A script is **repeatable, version-controllable, and
> automatable**. Running `psql -f kino.sql` recreates the entire database
> identically every time, with no risk of typos or skipped statements. The
> script can be committed to Git, reviewed, shared, and re-run in CI or on
> another server — making the setup reproducible and self-documenting.
>
> An interactive approach is still preferable for **exploration and
> debugging**: when you do not yet know the exact statements, want to inspect
> intermediate results, test a query, or react to errors step by step.
> Interactive `psql` gives immediate feedback and lets you adjust as you go,
> which is ideal during development before the final statements are captured
> into a script.

---

## Further Reading

- [PostgreSQL 16 – `CREATE ROLE`](https://www.postgresql.org/docs/current/sql-createrole.html)
- [PostgreSQL 16 – `COPY`](https://www.postgresql.org/docs/current/sql-copy.html)
- [PostgreSQL 16 – `psql` Reference](https://www.postgresql.org/docs/current/app-psql.html)
- [PostgreSQL 16 – Data Types](https://www.postgresql.org/docs/current/datatype.html)
- [PostgreSQL 16 – Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- Lecture 06 handout
