- [5. Dynamisches SQL](#5-dynamisches-sql)
  - [5.1 Dynamischer COUNT auf einer Tabelle](#51-dynamischer-count-auf-einer-tabelle)
  - [5.2 Dynamische Abfrage mit Parameter (EXECUTE … USING)](#52-dynamische-abfrage-mit-parameter-execute--using)
  - [5.3 Dynamische Spaltenauswahl](#53-dynamische-spaltenauswahl)

Bearbeitungszeit: 60 Minuten

# 5. Dynamisches SQL

In dieser Aufgabe lernst du, dynamisches SQL in PL/pgSQL-Funktionen einzusetzen.
Du arbeitest mit der PostgreSQL Northwind Database und nutzt dabei:

- EXECUTE
- EXECUTE … USING
- dynamische Tabellen- und Spaltennamen
- sichere Techniken zur Vermeidung von SQL Injection

## 5.1 Dynamischer COUNT auf einer Tabelle

**Aufgabe:**

Erstelle eine Funktion count_rows(p_table_name text), die die Anzahl der Datensätze einer beliebigen Tabelle zurückgibt.

**Anforderungen:**

- Der Tabellenname wird als Parameter übergeben
- Es muss dynamisches SQL verwendet werden
- Die Funktion soll einen integer zurückgeben

**Erwartetes Ergebnis**

```sql
SELECT count_rows('orders');
```

| count_rows |
| ---------: |
| 830 |

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE FUNCTION count_rows(p_table_name text)
RETURNS integer AS $$
DECLARE
  result integer;
BEGIN
  EXECUTE format('SELECT count(*) FROM %I', p_table_name)
  INTO result;

  RETURN result;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 5.2 Dynamische Abfrage mit Parameter (EXECUTE … USING)

**Aufgabe:**

Erstelle eine Funktion get_orders_by_customer(p_customer_id text), die alle Bestellungen eines Kunden zurückgibt.

**Hinweise:**

- Tabelle: `orders`
- Filter auf `customer_id`
- Rückgabe soll `order_id`, `order_date`, `freight` enthalten
- Verwende EXECUTE … USING

**Erwartetes Ergebnis**

```sql
SELECT * FROM get_orders_by_customer('ALFKI');
```

| order_id | order_date | freight |
| -------: | ---------- | ------: |
| 10643	| 1997-08-25 | 29.46 |
| 10692 | 1997-10-03 | 61.02 |

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE FUNCTION get_orders_by_customer(p_customer_id text)
RETURNS TABLE (
  order_id smallint,
  order_date date,
  freight real
) AS $$
BEGIN
  RETURN QUERY
  EXECUTE
    'SELECT order_id, order_date, freight
     FROM orders
     WHERE customer_id = $1'
  USING p_customer_id;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 5.3 Dynamische Spaltenauswahl

**Aufgabe:**
Erstelle eine Funktion get_column_values(p_table text, p_column text), die alle Werte einer beliebigen Spalte aus einer Tabelle zurückgibt.

**Anforderungen:**

- Tabellen- und Spaltenname sind Parameter
- Verwende format() mit %I
- Rückgabe als SETOF text

**Erwartetes Ergebnis**

```sql
SELECT * FROM get_column_values('customers', 'country');
```

| get_column_values |
| ----------------- |
| Germany |
| USA |
| France |
| ... |

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE FUNCTION get_column_values(
  p_table text,
  p_column text
)
RETURNS SETOF text AS $$
BEGIN
  RETURN QUERY
  EXECUTE format(
    'SELECT %I::text FROM %I',
    p_column,
    p_table
  );
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 5.4 Sicheres dynamisches SQL (SQL Injection vermeiden)

**Aufgabe:**

Erstelle eine Funktion safe_order_lookup(p_column text, p_value text), die Bestellungen anhand einer dynamischen Spalte, aber sicher filtert.

**Vorgaben:**

- Tabelle: `orders`
- Erlaubte Spalten: `customer_id`, `ship_country`
- Ungültige Spalten sollen mit einer Exception abgelehnt werden
- Filterwert muss über `USING` gesetzt werden

**Erwartetes Ergebnis**

```sql
SELECT * FROM safe_order_lookup('ship_country', 'Germany');
```

| order_id | ship_country |
| -------: | ------------ |
| 10267 | Germany |
| ... | ... |

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE FUNCTION safe_order_lookup(
  p_column text,
  p_value text
)
RETURNS TABLE (
  order_id smallint,
  ship_country character varying(15)
) AS $$
DECLARE
  sql text;
BEGIN
  IF p_column NOT IN ('customer_id', 'ship_country') THEN
    RAISE EXCEPTION 'Spalte % ist nicht erlaubt', p_column;
  END IF;

  sql := format(
    'SELECT order_id, ship_country
     FROM orders
     WHERE %I = $1',
    p_column
  );

  RETURN QUERY
  EXECUTE sql USING p_value;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>
