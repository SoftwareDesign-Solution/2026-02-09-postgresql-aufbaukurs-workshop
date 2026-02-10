- [2. Funktionen & Prozeduren](#2-funktionen--prozeduren)
  - [2.1 SCALAR-Funktion (RETURNS NUMERIC)](#21-scalar-funktion-returns-numeric)
  - [2.2 RETURNS TABLE Funktion](#22-returns-table-funktion)
  - [2.3 Funktionsattribute (STABLE)](#23-funktionsattribute-stable)
  - [2.4 Funktionsoverloading](#24-funktionsoverloading)
  - [2.5 PROCEDURE mit Datenänderung](#25-procedure-mit-datenänderung)

Bearbeitungszeit: 60 Minuten

# 2. Funktionen & Prozeduren

In dieser Aufgabe arbeitest du mit der Northwind-Datenbank und erstellst verschiedene Funktionen und eine Prozedur, um typische Geschäftslogik serverseitig abzubilden.

## 2.1 SCALAR-Funktion (RETURNS NUMERIC)

Erstellen Sie eine Funktion:

- Name: `order_total_with_vat(order_id INT)`
- Rückgabewert: Gesamtbetrag der Bestellung inkl. 19 % MwSt
- Nutzen Sie dazu:
  - Tabelle `order_details`
  - Berechnung: `unit_price * quantity * (1 - discount)`
- Rückgabetyp: NUMERIC

Testen Sie die Funktion mit einer existierenden order_id.

**Erwartetes Ergebnis bei order_id 10248**

| order_total_with_vat |
| -------------------: |
| 523.59999773025469 |

<details>
<summary>Show solution</summary>
<p>

**Funktion `order_total_with_vat` LANGUAGE plpqsql**

```sql
CREATE OR REPLACE FUNCTION order_total_with_vat(p_order_id INT)
RETURNS NUMERIC
AS $$
DECLARE
  total NUMERIC;
BEGIN
  SELECT SUM(unit_price * quantity * (1 - discount))
  INTO total
  FROM order_details
  WHERE order_id = p_order_id;

  RETURN total * 1.19;
END;
$$ LANGUAGE plpgsql;

SELECT order_total_with_vat(10248);
```

**Funktion `order_total_with_vat` LANGUAGE sql**

```sql
CREATE OR REPLACE FUNCTION order_total_with_vat(p_order_id INT)
RETURNS NUMERIC
AS $$
  SELECT
    SUM(unit_price * quantity * (1 - discount)) * 1.19
  FROM order_details
  WHERE order_id = p_order_id;
$$ LANGUAGE sql;

SELECT order_total_with_vat(10248);
```

</p>
</details>

## 2.2 RETURNS TABLE Funktion

Erstellen Sie eine Funktion:

- Name: `orders_by_customer(p_customer_id TEXT)`
- Rückgabe:
  - `order_id smallint`
  - `order_date DATE`
  - `ship_country character varying(15)`
- Es sollen alle Bestellungen eines Kunden zurückgegeben werden

Nutzen Sie `RETURN QUERY`.

**Erwartetes Ergebnis**

| order_id | order_date | ship_country |
| -------: | ---------- | ------------ |
| 10643 | 1997-08-25 | Germany |
| 10692 | 1997-10-03 | Germany |
| 10702 | 1997-10-13 | Germany |
| 10835 | 1998-01-15 | Germany |
| 10952 | 1998-03-16 | Germany |
| 11011 | 1998-04-09 | Germany |

<details>
<summary>Show solution</summary>
<p>

**Funktion `orders_by_customer`**

```sql
CREATE OR REPLACE FUNCTION orders_by_customer(p_customer_id TEXT)
RETURNS TABLE(order_id SMALLINT, order_date DATE, ship_country character varying(15))
AS $$
BEGIN
  RETURN QUERY
  SELECT o.order_id, o.order_date, o.ship_country
  FROM orders o
  WHERE o.customer_id = p_customer_id;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM orders_by_customer('ALFKI');
```

</p>
</details>

## 2.3 Funktionsattribute (STABLE)

Erweitern Sie die Funktion aus **Teilaufgabe 2.2**:

- Kennzeichnen Sie die Funktion als **STABLE**
- Begründen Sie kurz, warum STABLE hier sinnvoll ist

**Erwartetes Ergebnis**

| order_id | order_date | ship_country |
| -------: | ---------- | ------------ |
| 10643 | 1997-08-25 | Germany |
| 10692 | 1997-10-03 | Germany |
| 10702 | 1997-10-13 | Germany |
| 10835 | 1998-01-15 | Germany |
| 10952 | 1998-03-16 | Germany |
| 11011 | 1998-04-09 | Germany |

<details>
<summary>Show solution</summary>
<p>

**Funktion `orders_by_customer`**

```sql
CREATE OR REPLACE FUNCTION orders_by_customer(p_customer_id TEXT)
RETURNS TABLE(order_id SMALLINT, order_date DATE, ship_country character varying(15))
STABLE
AS $$
BEGIN
  RETURN QUERY
  SELECT o.order_id, o.order_date, o.ship_country
  FROM orders o
  WHERE o.customer_id = p_customer_id;
END;
$$ LANGUAGE plpgsql;
```

**Begründung:**

- Die Funktion liest nur Daten
- Das Ergebnis bleibt innerhalb einer Query konstant
- Keine Datenänderungen → STABLE ist korrekt

</p>
</details>

## 2.4 Funktionsoverloading

Erstellen Sie eine zweite Funktion mit gleichem Namen `order_total_with_vat`:

- Parameter:
  - `prder_id INT``
  - `vat_rate NUMERIC`
- Rückgabewert:
  - Gesamtbetrag mit **frei übergebener MwSt**
- Testen Sie beide Varianten der Funktion

**Erwartetes Ergebnis bei order_id 10248**

| order_total_with_vat |
| -------------------: |
| 523.59999773025469 |

| order_total_with_vat |
| -------------------: |
| 470.79999795913657 |

<details>
<summary>Show solution</summary>
<p>

**Funktion `order_total_with_vat`**

```sql
CREATE OR REPLACE FUNCTION order_total_with_vat(
  p_order_id INT,
  p_vat_rate NUMERIC
)
RETURNS NUMERIC
AS $$
DECLARE
  total NUMERIC;
BEGIN
  SELECT SUM(unit_price * quantity * (1 - discount))
  INTO total
  FROM order_details
  WHERE order_id = p_order_id;

  RETURN total * (1 + p_vat_rate);
END;
$$ LANGUAGE plpgsql;

SELECT order_total_with_vat(10248);
SELECT order_total_with_vat(10248, 0.07);
```

</p>
</details>

## 2.5 PROCEDURE mit Datenänderung

Erstellen Sie eine Prozedur:

- Name: `increase_product_price(category_id INT, percent NUMERIC)``
- Aufgabe:
  - Erhöht den Preis (`unit_price`) aller Produkte einer Kategorie
- Nutzen Sie:
  - Tabelle `products`
- Führen Sie am Ende einen `COMMIT` aus

<details>
<summary>Show solution</summary>
<p>

**Prozedur `increase_product_price`**

```sql
CREATE OR REPLACE PROCEDURE increase_product_price(
  p_category_id INT,
  p_percent NUMERIC
)
AS $$
BEGIN
  UPDATE products
  SET unit_price = unit_price * (1 + p_percent)
  WHERE category_id = p_category_id;

  COMMIT;
END;
$$ LANGUAGE plpgsql;

CALL increase_product_price(1, 0.10);
```

</p>
</details>
