# Hands-On-Lab-Audit-Logging-Category-Trees-and-Safe-Migrations
Step 1: Build a Reusable Audit Log

binabdallah@penguin:~$ sudo -u postgres psql
psql (15.19 (Debian 15.19-0+deb12u1))
Type "help" for help.

postgres=# CREATE TABLE audit_log (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tbl TEXT,
  op TEXT,
  old_row JSONB,
  new_row JSONB,
  changed_by TEXT DEFAULT current_user,
  at TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE
postgres=# CREATE TABLE audit_log (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tbl TEXT,
  op TEXT,
  old_row JSONB,
  new_row JSONB,
  changed_by TEXT DEFAULT current_user,
  at TIMESTAMPTZ DEFAULT now()
);
ERROR:  relation "audit_log" already exists
postgres=# CREATE FUNCTION audit() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log(tbl, op, old_row, new_row)
  VALUES (
    TG_TABLE_NAME,
    TG_OP,
    to_jsonb(OLD),
    to_jsonb(NEW)
  );
postgres$# RETURN COALESCE(NEW, OLD);
END;
postgres$# $$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# CREATE TRIGGER trg_audit
AFTER INSERT OR UPDATE OR DELETE
ON students
FOR EACH ROW
EXECUTE FUNCTION audit();
CREATE TRIGGER
postgres=# 

Step 2: Watch the Audit Log Work
postgres=# UPDATE students SET name = 'Kofi M.' WHERE id = 1;
DELETE FROM students WHERE id = 3;

SELECT
  tbl,
  op,
  old_row->>'name' AS was,
  new_row->>'name' AS now,
  at
FROM audit_log
ORDER BY at DESC;
UPDATE 1
DELETE 0
   tbl    |   op   |  was  |   now   |              at               
----------+--------+-------+---------+-------------------------------
 students | UPDATE | Alice | Kofi M. | 2026-09-23 08:56:21.430772+03
(1 row)

postgres=# 

Step 3: Model a Category Tree
postgres=# CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name TEXT,
  parent_id INT REFERENCES categories(id)
);
CREATE TABLE
postgres=#  
INSERT INTO categories (name, parent_id) VALUES
  ('Electronics', NULL),
  ('Computers', 1),
  ('Laptops', 2),
  ('Phones', 1);
INSERT 0 4
postgres=# WITH RECURSIVE tree AS (
  SELECT id, name, parent_id, 0 AS depth
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  SELECT c.id, c.name, c.parent_id, t.depth + 1
  FROM categories c
  JOIN tree t ON c.parent_id = t.id
)
SELECT repeat('  ', depth) || name AS tree
FROM tree
ORDER BY depth, name;
    tree     
-------------
 Electronics
   Computers
   Phones
     Laptops
(4 rows)

postgres=# 

Step 4: Run Versioned Migrations with Flyway

postgres=# flyway -url=jdbc:postgresql://localhost/bootcamp -user=postgres migrate
flyway info

Step 5: Apply Least‑Privilege Security
postgres=# CREATE ROLE app_read;
CREATE ROLE
postgres=# CREATE ROLE app_write;
CREATE ROLE
postgres=# GRANT CONNECT ON DATABASE bootcamp TO app_read, app_write;
GRANT
postgres=# GRANT USAGE ON SCHEMA public TO app_read, app_write;
GRANT
postgres=#  GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
GRANT
postgres=# GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public TO app_write;
GRANT
postgres=# CREATE USER api
