- [4. Set-basierte Verarbeitung in Funktionen](#4-set-basierte-verarbeitung-in-funktionen)
  - [4.1 Feld `status` anlegen](#41-feld-status-anlegen)
  - [4.2 Set-basiertes UPDATE in einer Funktion](#42-set-basiertes-update-in-einer-funktion)
  - [4.3 INSERT … SELECT in einer Funktion](#43-insert--select-in-einer-funktion)
  - [4.4 UPDATE mit RETURNING](#44-update-mit-returning)

Bearbeitungszeit: 90 Minuten

# 4. Set-basierte Verarbeitung in Funktionen

In dieser Aufgabe sollst du lernen, mehrere Datensätze innerhalb einer Funktion set-basiert zu verarbeiten – ohne Schleifen.
Als Datenbasis dient die PostgreSQL Northwind Database.

Ziel ist es, typische Batch-Operationen (UPDATE, INSERT, RETURNING) effizient in Funktionen umzusetzen.

## 4.1 Feld `status` anlegen

In der Northwind-Datenbank existiert in der Tabelle `orders` noch kein Feld zur Abbildung des Bestellstatus.

Lege daher einen **ENUM-Typ** für den Status an und verwende diesen in der Tabelle `orders`.

**Anforderungen**

- ENUM-Typ mit dem Namen `order_status`
- Erlaubte Werte:
  - `'New'`
  - `'Processed'`
  - `'Closed'`
  - `'Archived'`
- Neues Feld `status` in der Tabelle `orders`
- Standardwert: `'New'`
- Bereits vorhandene Bestellungen sollen automatisch den Status `'New'` erhalten

**Hinweise**

- ENUM-Typen sind PostgreSQL-spezifisch
- Die Aufgabe ist Voraussetzung für alle folgenden Teilaufgaben

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE TYPE order_status AS ENUM (
  'New',
  'Processed',
  'Closed',
  'Archived'
);
```

```sql
ALTER TABLE orders
ADD COLUMN status order_status NOT NULL DEFAULT 'New';
```

</p>
</details>

## 4.2 Set-basiertes UPDATE in einer Procedure

Erstelle ein Procedure `archive_old_orders()`, die alle Bestellungen (`orders`) auf den Status `'Archived'` setzt,

- deren `order_date` **älter als 5 Jahre** ist
- und deren aktueller Status **nicht bereits** `'Archived'` ist.

Die Funktion soll:

- **keine Schleifen** enthalten
- `void` zurückgeben

**Hinweise**

- Tabelle: `orders`
- Spalte für Status: `status`

Aktuelles Datum: CURRENT_DATE

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE PROCEDURE archive_old_orders()
RETURNS void AS $$
BEGIN
  UPDATE orders
  SET status = 'Archived'
  WHERE order_date < CURRENT_DATE - INTERVAL '5 years'
    AND status <> 'Archived';
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 4.3 INSERT … SELECT in einer Funktion

Erstelle eine Tabelle `customer_order_stats` mit folgenden Spalten:

- `customer_id`
- `order_count`
- `last_order_date`

Anschließend erstelle eine Funktion `refresh_customer_order_stats()`, die:

- alle Kunden aus `orders` aggregiert
- die Ergebnisse **set-basiert** in `customer_order_stats` schreibt
- bestehende Daten vorher vollständig löscht

**Hinweise**

- Verwende `COUNT(*)` und `MAX(order_date)`
- Keine Schleifen verwenden

<details>
<summary>Show solution</summary>
<p>

**Tabelle `customer_order_stats`**

```sql
CREATE TABLE customer_order_stats (
  customer_id text,
  order_count int,
  last_order_date date
);
```

**Funktion `refresh_customer_order_stats`**

```sql
CREATE OR REPLACE FUNCTION refresh_customer_order_stats()
RETURNS void AS $$
BEGIN
  DELETE FROM customer_order_stats;

  INSERT INTO customer_order_stats (customer_id, order_count, last_order_date)
  SELECT
    customer_id,
    COUNT(*),
    MAX(order_date)
  FROM orders
  GROUP BY customer_id;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>

## 4.4 UPDATE mit RETURNING

Erstelle eine Funktion `close_orders_before(date)`, die:

- alle Bestellungen mit `order_date` **vor dem übergebenen Datum**
- auf den Status `'Closed'` setzt
- und **alle betroffenen Bestell-IDs zurückgibt**

**Anforderungen**

- Set-basiertes `UPDATE`
- Nutzung von `RETURNING`
- Rückgabewert: Liste von IDs

**Erwartetes Ergebnis**

| order_id |
| -------: |
| 10248 |
| ... |

<details>
<summary>Show solution</summary>
<p>

**Funktion `close_orders_before`**

```sql
CREATE OR REPLACE FUNCTION close_orders_before(p_date date)
RETURNS TABLE(order_id smallint) AS $$
BEGIN
  RETURN QUERY
  UPDATE orders o
  SET status = 'Closed'
  WHERE o.order_date < p_date
  RETURNING o.order_id;
END;
$$ LANGUAGE plpgsql;
```

</p>
</details>
