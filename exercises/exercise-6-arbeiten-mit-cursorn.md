- [6. Arbeiten mit Cursorn](#6-arbeiten-mit-cursorn)
  - [6.1 Cursor: Öffnen, Lesen, Schließen](#61-cursor-öffnen-lesen-schließen)

Bearbeitungszeit: 60 Minuten

# 6. Arbeiten mit Cursorn

In dieser Aufgabe übst du das Arbeiten mit Cursorn in PostgreSQL (PL/pgSQL) anhand der Northwind-Datenbank.

Du behandelst:

- Cursor deklarieren/öffnen/lesen/schließen
- Cursor in Schleifen
- SCROLL vs. NO SCROLL (Blättern)
- REFCURSOR als Rückgabe (Resultset „später“ fetchbar)

## 6.1 Cursor: Öffnen, Lesen, Schließen

**Aufgabe:**

Schreibe einen DO $$ ... $$-Block in PL/pgSQL, der:

1. Einen NO SCROLL Cursor über die letzten Bestellungen (nach order_date) deklariert.
2. Den Cursor öffnet.
3. Die ersten 5 Bestellungen mit FETCH NEXT liest.
4. Pro gelesener Zeile eine Ausgabe mit RAISE NOTICE erzeugt, z. B.:\
  Order <order_id> | Customer <customer_id> | Date <order_date>

Den Cursor schließt.

Ergebnis: Du siehst 5 Zeilen als NOTICE.

**Erwartetes Ergebnis**

```text
NOTICE:  Order 11074 | Customer SIMOB | Date 1998-05-06
NOTICE:  Order 11077 | Customer RATTC | Date 1998-05-06
NOTICE:  Order 11076 | Customer BONAP | Date 1998-05-06
NOTICE:  Order 11075 | Customer RICSU | Date 1998-05-06
NOTICE:  Order 11071 | Customer LILAS | Date 1998-05-05
```

<details>
<summary>Show solution</summary>
<p>

```sql
DO $$
DECLARE
  v_order_id     int;
  v_customer_id  text;
  v_order_date   date;

  cur_orders NO SCROLL CURSOR FOR
    SELECT o.order_id, o.customer_id, o.order_date
    FROM orders o
    ORDER BY o.order_date DESC;
BEGIN
  OPEN cur_orders;

  FOR i IN 1..5 LOOP
    FETCH NEXT FROM cur_orders INTO v_order_id, v_customer_id, v_order_date;
    EXIT WHEN NOT FOUND;

    RAISE NOTICE 'Order % | Customer % | Date %', v_order_id, v_customer_id, v_order_date;
  END LOOP;

  CLOSE cur_orders;
END $$;
```

</p>
</details>

## 6.2 Cursor in Schleife: Status-Update je Bestellung

**Aufgabe:**

1. Schreibe eine Prozedur close_small_orders(p_max_total numeric), die:
    - per Cursor alle Bestellungen mit `status = 'New'` durchläuft,
    - je Bestellung die Gesamtsumme berechnet: \
    `SUM(unit_price * quantity * (1 - discount))` aus `order_details`,
    - wenn die Summe < `p_max_total` ist, setze `orders.status = 'Closed'`,
    - gib bei jedem Update ein RAISE NOTICE aus: \
    `Closed order <order_id> total=<total>`

Verwende für den Parameter `p_max_total` den festen Wert 50.

> Ziel: Cursor + Schleife + „pro Datensatz Business-Logik“.

**Erwartetes Ergebnis**

```text
NOTICE:  Closed order 10271 total=48
NOTICE:  Closed order 10422 total=49.7999992370605
NOTICE:  Closed order 10586 total=23.799999833107
NOTICE:  Closed order 10602 total=48.75
NOTICE:  Closed order 10674 total=45
NOTICE:  Closed order 10767 total=28
NOTICE:  Closed order 10782 total=12.5
NOTICE:  Closed order 10807 total=18.3999996185303
NOTICE:  Closed order 10815 total=40
NOTICE:  Closed order 10883 total=36
NOTICE:  Closed order 10898 total=30
NOTICE:  Closed order 10900 total=33.75
NOTICE:  Closed order 11051 total=35.9999998658895
NOTICE:  Closed order 11057 total=45
```

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE PROCEDURE close_small_orders(p_max_total numeric)
LANGUAGE plpgsql
AS $$
DECLARE
  v_order_id int;
  v_total    numeric;

  cur_orders NO SCROLL CURSOR FOR
    SELECT o.order_id
    FROM orders o
    WHERE o.status = 'New'
    ORDER BY o.order_id;
BEGIN
  OPEN cur_orders;

  LOOP
    FETCH NEXT FROM cur_orders INTO v_order_id;
    EXIT WHEN NOT FOUND;

    SELECT COALESCE(SUM(od.unit_price * od.quantity * (1 - od.discount)), 0)
    INTO v_total
    FROM order_details od
    WHERE od.order_id = v_order_id;

    IF v_total < p_max_total THEN
      UPDATE orders
      SET status = 'Closed'
      WHERE order_id = v_order_id;

      RAISE NOTICE 'Closed order % total=%', v_order_id, v_total;
    END IF;
  END LOOP;

  CLOSE cur_orders;
END $$;

-- Beispielaufruf:
-- CALL close_small_orders(50.00);
```

</p>
</details>

## 6.3 SCROLL Cursor: Blättern (NEXT/PRIOR/ABSOLUTE)

**Aufgabe:**

Schreibe einen DO $$ ... $$-Block, der:

1. Einen SCROLL Cursor über `products` deklariert (z. B. `product_id`, `product_name`, `unit_price` nach `product_id` sortiert).
2. Den Cursor öffnet und dann:
    - `FETCH NEXT` (1. Produkt) ausgibt
    - `FETCH ABSOLUTE` 10 (10. Produkt) ausgibt
    - `FETCH PRIOR` (9. Produkt) ausgibt
3. Den Cursor schließt.

Ergebnis: Du siehst 3 `NOTICE`-Ausgaben mit unterschiedlichen Produkten.

**Erwartetes Ergebnis**

```text
NOTICE:  NEXT: id=1 name=Chai price=19.8
NOTICE:  ABS 10: id=10 name=Ikura price=31
NOTICE:  PRIOR: id=9 name=Mishi Kobe Niku price=97
```

<details>
<summary>Show solution</summary>
<p>

```sql
DO $$
DECLARE
  v_id    int;
  v_name  text;
  v_price numeric;

  cur_products SCROLL CURSOR FOR
    SELECT p.product_id, p.product_name, p.unit_price
    FROM products p
    ORDER BY p.product_id;
BEGIN
  OPEN cur_products;

  FETCH NEXT FROM cur_products INTO v_id, v_name, v_price;
  RAISE NOTICE 'NEXT: id=% name=% price=%', v_id, v_name, v_price;

  FETCH ABSOLUTE 10 FROM cur_products INTO v_id, v_name, v_price;
  RAISE NOTICE 'ABS 10: id=% name=% price=%', v_id, v_name, v_price;

  FETCH PRIOR FROM cur_products INTO v_id, v_name, v_price;
  RAISE NOTICE 'PRIOR: id=% name=% price=%', v_id, v_name, v_price;

  CLOSE cur_products;
END $$;
```

</p>
</details>

## 6.4 REFCURSOR zurückgeben: „Top Orders“ pro Kunde

**Aufgabe:**

Erstelle eine Funktion `get_customer_orders_cursor(p_customer_id text)` die:

1. Einen `REFCURSOR` öffnet für eine Abfrage, die Bestellungen des Kunden zeigt:
    - `order_id`
    - `order_date`
    - `freight`
2. Sortierung: order_date DESC
3. Den Cursor zurückgibt.

Teste die Funktion in einer Transaktion:

- BEGIN;
- Cursor-Funktion aufrufen
- FETCH 5 FROM <cursorname>;
- --COMMIT; -- Auskommentiert lassen, da sonst das Ergebnis nicht angezeigt wird

**Erwartetes Ergebnis**

| order_id | order_date | freight |
| -------: | ---------- | ------: |
| 11011 | 1998-04-09 | 1.21 |
| 10952 | 1998-03-16 | 40.42 |
| 10835 | 1998-01-15 | 69.53 |
| 10702 | 1997-10-13 | 23.94 |
| 10692 | 1997-10-03 | 61.02 |

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE FUNCTION get_customer_orders_cursor(p_customer_id text)
RETURNS refcursor
LANGUAGE plpgsql
AS $$
DECLARE
  c refcursor := 'cust_orders';
BEGIN
  OPEN c FOR
    SELECT o.order_id, o.order_date, o.freight
    FROM orders o
    WHERE o.customer_id = p_customer_id
    ORDER BY o.order_date DESC;

  RETURN c;
END $$;

-- Test (Beispiel-Customer-ID ggf. anpassen)
BEGIN;
SELECT get_customer_orders_cursor('ALFKI');
FETCH 5 FROM cust_orders;
--COMMIT;
```

</p>
</details>
