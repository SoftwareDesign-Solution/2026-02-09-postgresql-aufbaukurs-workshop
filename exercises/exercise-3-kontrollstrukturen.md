- [3. Kontrollstrukturen](#3-kontrollstrukturen)
  - [3.1 CASE-Ausdruck für Versandkosten-Klassifikation](#31-case-ausdruck-für-versandkosten-klassifikation)
  - [3.2 IF / ELSIF: Rabattlogik pro Bestellung](#32-if--elsif-rabattlogik-pro-bestellung)
  - [3.3 FOR-Loop: Top-N Kunden nach Umsatz (RETURN NEXT)](#33-for-loop-top-n-kunden-nach-umsatz-return-next)
  - [3.4 RETURN QUERY: Bestellungen mit klassifiziertem Freight](#34-return-query-bestellungen-mit-klassifiziertem-freight)
  - [3.5 Bonusaufgabe: LOOP + EXIT WHEN (Robuste Eingabe)](#35-bonusaufgabe-loop--exit-when-robuste-eingabe)

Bearbeitungszeit: 90 Minuten

# 3. Kontrollstrukturen

In dieser Aufgabe nutzt du Kontrollstrukturen in PL/pgSQL (IF/ELSIF/ELSE, CASE, LOOP/FOR, RETURN NEXT/RETURN QUERY) auf Basis der PostgreSQL Northwind Database.

Ziel: Du baust mehrere Funktionen, die typische Geschäftslogik direkt in der Datenbank abbilden (Rabatte, Bewertung von Kunden, Iteration über Datensätze).

## 3.1 CASE-Ausdruck für Versandkosten-Klassifikation

Erstelle eine Funktion `shipping_bucket(freight real)` die einen Text zurückgibt und teste diese Funktion:

- `freight < 20` → `'LOW'`
- `freight >= 20 AND freight < 60` → `'MEDIUM'`
- `freight >= 60` → `'HIGH'`

Hinweis: Nutze **CASE** (nicht IF).

**Erwartetes Ergebnis**

| freight | shipping_bucket |
| ------: | --------------- |
| 10 | LOW |
| 50 | MEDIUM |
| 80 | HIGH |

<details>
<summary>Show solution</summary>
<p>

**Funktion `shipping_bucket`**

```sql
CREATE OR REPLACE FUNCTION shipping_bucket(freight real)
RETURNS text
AS $$
BEGIN
  RETURN CASE
    WHEN freight < 20 THEN 'LOW'
    WHEN freight < 60 THEN 'MEDIUM'
    ELSE 'HIGH'
  END;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 3.2 IF / ELSIF: Rabattlogik pro Bestellung

Erstelle eine Funktion `order_discount(order_id int)` die einen **Rabattfaktor** (numeric) zurückgibt:

- Gesamtwert der Bestellung < **100** → `1.00`
- **100 bis < 500** → `0.95`
- **>= 500** → `0.90`

Berechne dazu den Gesamtwert über `order_details` (üblich in Northwind: `unit_price * quantity * (1 - discount)`) und teste die Funktion.

**Erwartetes Ergebnis**

| p_order_id | order_discount |
| ---------: | -------------: |
| 10248 | 0.95 |

<details>
<summary>Show solution</summary>
<p>

**Funktion `order_discount`**

```sql
CREATE OR REPLACE FUNCTION order_discount(p_order_id int)
RETURNS numeric
AS $$
DECLARE
  total_value numeric;
BEGIN
  SELECT COALESCE(SUM(od.unit_price * od.quantity * (1 - od.discount)), 0)
    INTO total_value
  FROM order_details od
  WHERE od.order_id = p_order_id;

  IF total_value < 100 THEN
    RETURN 1.00;
  ELSIF total_value < 500 THEN
    RETURN 0.95;
  ELSE
    RETURN 0.90;
  END IF;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 3.3 FOR-Loop: Top-N Kunden nach Umsatz (RETURN NEXT)

Erstelle eine Funktion `top_customers_by_revenue(n int)` die **SETOF** zurückgibt mit:

- `customer_id`
- `revenue`

Vorgaben:

- Berechne Umsatz über Orders + Order Details
- Gib die **Top n** Kunden zurück
- Nutze eine **FOR-Schleife über ein SELECT** und gib Zeilen mit **RETURN NEXT** zurück

**Erwartetes Ergebnis**

| top_customers_by_revenue |
| ------------------------ |
| QUICK","110277.305030394 |
| ERNSH","104874.978143677 |

<details>
<summary>Show solution</summary>
<p>

**Funktion `top_customers_by_revenue`**

```sql
CREATE OR REPLACE FUNCTION top_customers_by_revenue(p_n int)
RETURNS TABLE(customer_id text, revenue numeric)
AS $$
DECLARE
  rec RECORD;
BEGIN
  FOR rec IN
    SELECT o.customer_id,
           SUM(od.unit_price * od.quantity * (1 - od.discount)) AS revenue
    FROM orders o
    JOIN order_details od ON od.order_id = o.order_id
    GROUP BY o.customer_id
    ORDER BY revenue DESC
    LIMIT p_n
  LOOP
    customer_id := rec.customer_id;
    revenue := rec.revenue;
    RETURN NEXT;
  END LOOP;

  RETURN;
END;
$$ LANGUAGE plpgsql;

SELECT top_customers_by_revenue(2);
```

</p>
</details>

## 3.4 RETURN QUERY: Bestellungen mit klassifiziertem Freight

Erstelle eine Funktion `orders_with_shipping_class(from_date date, to_date date)` die eine Ergebnismenge zurückgibt:

| Spalte | Datentyp |
| ------ | -------- |
| `order_id` | `SMALLINT` |
| `order_date` | `DATE` |
| `freight` | `REAL` |
| `shipping_class` (über `shipping_bucket(freight)`) | `TEXT` |

Vorgaben:

- Nutze **RETURN QUERY**
- Filtere nach `order_date` im Zeitraum

**Erwartetes Ergebnis**

| orders_with_shipping_class |
| ------------------------ |
| 10248","1996-07-04","32.38","MEDIUM" |
| 10249","1996-07-05","11.61","LOW |

<details>
<summary>Show solution</summary>
<p>

**Funktion `orders_with_shipping_class`**

```sql
CREATE OR REPLACE FUNCTION orders_with_shipping_class(from_date date, to_date date)
RETURNS TABLE(order_id SMALLINT, order_date date, freight real, shipping_class text)
AS $$
BEGIN
  RETURN QUERY
  SELECT o.order_id,
         o.order_date::date,
         o.freight,
         shipping_bucket(o.freight) AS shipping_class
  FROM orders o
  WHERE o.order_date::date BETWEEN from_date AND to_date
  ORDER BY o.order_date;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 3.5 Bonusaufgabe: LOOP + EXIT WHEN (Robuste Eingabe)

Erstelle eine Funktion `safe_top_customers(n int)` die sicherstellt:

- Wenn `n <= 0`, dann setze `n := 5`
- Wenn `n > 50`, dann setze `n := 50`

Nutze dafür absichtlich eine **LOOP** mit `EXIT WHEN` und rufe anschließend `top_customers_by_revenue(n)` per `RETURN QUERY` auf.

**Erwartetes Ergebnis**

| safe_top_customers |
| ------------------ |
| QUICK","110277.305030394 |

<details>
<summary>Show solution</summary>
<p>

**Funktion `safe_top_customers`**

```sql
CREATE OR REPLACE FUNCTION safe_top_customers(p_n int)
RETURNS TABLE(customer_id text, revenue numeric)
AS $$
BEGIN
  LOOP
    IF p_n <= 0 THEN
      p_n := 5;
    ELSIF p_n > 50 THEN
      p_n := 50;
    END IF;

    EXIT WHEN p_n BETWEEN 1 AND 50;
  END LOOP;

  RETURN QUERY
  SELECT * FROM top_customers_by_revenue(p_n);
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>
