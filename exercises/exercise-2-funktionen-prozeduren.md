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

<details>
<summary>Show solution</summary>
<p>

**Funktion `order_total_with_vat`**

```sql
CREATE OR REPLACE FUNCTION order_total_with_vat(p_order_id INT)
RETURNS NUMERIC
LANGUAGE plpgsql
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
$$;

SELECT order_total_with_vat(10248);
```

</p>
</details>

## 2.2 RETURNS TABLE Funktion

Erstellen Sie eine Funktion:

- Name: `orders_by_customer(customer_id TEXT)`
- Rückgabe:
  - `order_id INT`
  - `order_date DATE`
  - `ship_country TEXT`
- Es sollen alle Bestellungen eines Kunden zurückgegeben werden

Nutzen Sie `RETURN QUERY`.

<details>
<summary>Show solution</summary>
<p>

**Funktion `orders_by_customer`**

```sql
CREATE OR REPLACE FUNCTION orders_by_customer(p_customer_id TEXT)
RETURNS TABLE(order_id INT, order_date DATE, ship_country TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT o.order_id, o.order_date, o.ship_country
  FROM orders o
  WHERE o.customer_id = p_customer_id;
END;
$$;

SELECT * FROM orders_by_customer('ALFKI');
```

</p>
</details>

## 2.3 Funktionsattribute (STABLE)

Erweitern Sie die Funktion aus **Teilaufgabe 2.2**:

- Kennzeichnen Sie die Funktion als **STABLE**
- Begründen Sie kurz, warum STABLE hier sinnvoll ist

<details>
<summary>Show solution</summary>
<p>

**Funktion `orders_by_customer`**

```sql
CREATE OR REPLACE FUNCTION orders_by_customer(p_customer_id TEXT)
RETURNS TABLE(order_id INT, order_date DATE, ship_country TEXT)
STABLE
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT o.order_id, o.order_date, o.ship_country
  FROM orders o
  WHERE o.customer_id = p_customer_id;
END;
$$;
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
LANGUAGE plpgsql
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
$$;

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
LANGUAGE plpgsql
AS $$
BEGIN
  UPDATE products
  SET unit_price = unit_price * (1 + p_percent)
  WHERE category_id = p_category_id;

  COMMIT;
END;
$$;

CALL increase_product_price(1, 0.10);
```

</p>
</details>
