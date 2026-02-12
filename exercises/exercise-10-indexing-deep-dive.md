- [10. Indexing Deep Dive](#10-indexing-deep-dive)
  - [10.1 Index-Kandidaten identifizieren (EXPLAIN)](#101-index-kandidaten-identifizieren-explain)

Bearbeitungszeit: 60 Minuten

# 10. Indexing Deep Dive

In dieser Aufgabe analysierst du typische Query-Patterns in der PostgreSQL Northwind Database und leitest daraus passende Index-Strategien ab. Du misst die Auswirkungen mit EXPLAIN (ANALYZE, BUFFERS) und lernst, wann B-Tree, GIN, BRIN, Partial und Expression Indexe sinnvoll sind – inklusive Wartung (REINDEX/VACUUM) und HOT-Updates.

> Hinweis: Falls deine Northwind-Variante leicht andere Tabellennamen/Spalten hat, passe die Queries entsprechend an (z. B. orders, order_details, customers, products).

## 10.1 Index-Kandidaten identifizieren (EXPLAIN)

**Aufgabe:**

Führe die folgenden Queries jeweils mit EXPLAIN (ANALYZE, BUFFERS) aus und notiere:

- ob ein Seq Scan oder Index Scan verwendet wird
- geschätzte vs. tatsächliche Zeilen (rows)
- auffällige I/O (shared read blocks, hit blocks)

**Query A (Orders nach Datum filtern):**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date
FROM orders
WHERE order_date >= DATE '1997-01-01'
  AND order_date <  DATE '1998-01-01'
ORDER BY order_date;
```

**Query B (Order Details nach Produkt):**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, product_id, quantity, unit_price
FROM order_details
WHERE product_id = 42;
```

**Query C (Kunden nach Country/City):**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, company_name, country, city
FROM customers
WHERE country = 'Germany'
  AND city = 'Berlin';
```

<details>
<summary>Show solution</summary>
<p>

**Erwartung / Lösungshinweise:**

- Wenn du Seq Scan siehst, sind das potenzielle Index-Kandidaten (abhängig von Tabellengröße & Selektivität).
- A: Typischer Kandidat für B-Tree auf (order_date) (evtl. zusätzlich (order_date, customer_id) je nach Abfragen).
- B: Typischer Kandidat für B-Tree auf (product_id) in order_details.
- C: Kandidat für Composite B-Tree (country, city) oder je nach Abfragehäufigkeit (country) plus (city); oft ist (country, city) sinnvoll, wenn häufig zusammen gefiltert wird.

</p>
</details>

## 10.2 B-Tree Index anlegen & Wirkung messen

**Aufgabe:**

Lege passende B-Tree-Indexe an, um Query A und B aus 1.1 zu beschleunigen. Miss danach erneut mit EXPLAIN (ANALYZE, BUFFERS).

1. Index für Query A (Datum):

  ```sql
  CREATE INDEX idx_orders_order_date
  ON orders(order_date);
  ```

2. Index für Query B (Produkt):
  
  ```sql
  CREATE INDEX idx_order_details_product_id
  ON order_details(product_id);
  ```

3. Wiederhole die EXPLAINs aus 1.1 für Query A und B und vergleiche:

  - Plan-Typ (Index Scan/Bitmap Index Scan?)
  - Ausführungszeit
  - BUFFERS (Hits/Reads)

<details>
<summary>Show solution</summary>
<p>

**Erwartete Beobachtung:**

- Query A: häufig Index Scan oder Bitmap Index Scan + Sort-Optimierung (abhängig von Datenmenge). ORDER BY order_date profitiert oft deutlich.
- Query B: sehr häufig Index Scan oder Bitmap Index Scan auf order_details(product_id).

Tipp: Nach dem Index-Anlegen:

```sql
ANALYZE orders;
ANALYZE order_details;
```

Damit der Planner aktuelle Statistiken hat.

</p>
</details>

## 10.3 Partial Index für „offene“ Bestellungen

**Aufgabe:**

Viele Systeme fragen „aktive/offene“ Datensätze häufig ab (Status/Flag). Simuliere das in Northwind:

1. Prüfe, ob es eine Spalte gibt, die „offen“ abbilden kann (z. B. shipped_date IS NULL).
2. Optimiere folgende Query mit einem Partial Index:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date
FROM orders
WHERE shipped_date IS NULL
ORDER BY order_date DESC;
```

Erstelle einen passenden Partial Index.

<details>
<summary>Show solution</summary>
<p>

**Lösung:**

```sql
CREATE INDEX idx_orders_open_by_order_date
ON orders(order_date DESC)
WHERE shipped_date IS NULL;
```

**Warum so?**

- Der Index enthält nur „offene“ Orders ⇒ kleiner, schneller.
- ORDER BY order_date DESC kann direkt aus dem Index kommen.

Optional danach:

```sql
ANALYZE orders;
```

</p>
</details>

## 10.4 Expression Index (LOWER/ILIKE)

**Aufgabe:**

Case-insensitive Suchen auf Textspalten sind häufig teuer, wenn Funktionen im WHERE stehen.

1. Prüfe den Plan:
  
  ```sql
  EXPLAIN (ANALYZE, BUFFERS)
  SELECT customer_id, company_name
  FROM customers
  WHERE LOWER(company_name) = 'alfreds futterkiste';
  ```

2. Lege einen Expression Index an, damit die Suche indexfähig wird.

<details>
<summary>Show solution</summary>
<p>

Lösung:

```sql
CREATE INDEX idx_customers_lower_company_name
ON customers (LOWER(company_name));
```

Danach erneut messen:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, company_name
FROM customers
WHERE LOWER(company_name) = 'alfreds futterkiste';
```

**Hinweis**: Für %...%-Suchen ist ein B-Tree nicht ideal. Dann wäre eher pg_trgm sinnvoll (GIN/GiST), siehe nächste Teilaufgabe-Variante.

</p>
</details>

## 10.5 GIN Index für Full-Text Search

**Aufgabe:**

Baue eine einfache Volltextsuche auf products(product_name) oder customers(company_name) auf und beschleunige sie mit GIN.

1. Query ohne Index (messen):
  
  ```sql
  EXPLAIN (ANALYZE, BUFFERS)
  SELECT product_id, product_name
  FROM products
  WHERE to_tsvector('simple', product_name) @@ plainto_tsquery('simple', 'chocolate');
  ```

2. Lege einen GIN Index an und messe erneut.

<details>
<summary>Show solution</summary>
<p>

Lösung:

```sql
CREATE INDEX idx_products_product_name_fts
ON products
USING gin (to_tsvector('simple', product_name));
```

Danach erneut:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id, product_name
FROM products
WHERE to_tsvector('simple', product_name) @@ plainto_tsquery('simple', 'chocolate');
```

Erwartung: Plan wechselt zu Bitmap Index Scan/Index Scan (je nach Version/Stats).

</p>
</details>

## 10.6 BRIN Index für große, zeitlich sortierte Daten

**Aufgabe (konzeptionell mit Northwind):** 

Northwind ist oft nicht „riesig“. Du simulierst deshalb eine große Log-Tabelle und testest BRIN.

1. Erzeuge eine große Tabelle:
  
  ```sql
  DROP TABLE IF EXISTS demo_logs;
CREATE TABLE demo_logs (
  id bigserial PRIMARY KEY,
  created_at timestamp not null,
  payload text
);

INSERT INTO demo_logs (created_at, payload)
SELECT
  timestamp '2020-01-01' + (gs * interval '1 minute'),
  md5(gs::text)
FROM generate_series(1, 500000) gs;

ANALYZE demo_logs;
  ```

2. Miss Query ohne Index:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*)
FROM demo_logs
WHERE created_at >= timestamp '2020-06-01'
  AND created_at <  timestamp '2020-06-08';
```

3. Lege BRIN an und miss erneut.

<details>
<summary>Show solution</summary>
<p>

Lösung:

```sql
CREATE INDEX idx_demo_logs_created_at_brin
ON demo_logs
USING brin(created_at);

ANALYZE demo_logs;
```

Erwartung:

- Deutlich weniger „gescannte“ Blöcke als beim Seq Scan (je nach Cache).
- BRIN ist besonders stark bei natürlich sortierten Daten (Insert nach Zeit).

</p>
</details>

## 10.7 Teilaufgabe – HOT Updates verstehen (Indexe & Updates)

**Aufgabe:**

Teste, wie sich zusätzliche Indexe auf Updates auswirken (HOT-Update-Prinzip).

1. Erzeuge Demo-Tabelle:
  
  ```sql
  DROP TABLE IF EXISTS demo_hot;
CREATE TABLE demo_hot (
  id bigserial PRIMARY KEY,
  status text NOT NULL,
  note text
);

INSERT INTO demo_hot(status, note)
SELECT 'open', md5(gs::text)
FROM generate_series(1, 200000) gs;

ANALYZE demo_hot;
  ```

2. Update ohne zusätzlichen Index:

```sql
UPDATE demo_hot
SET note = note || '_x'
WHERE id BETWEEN 1 AND 50000;
```

3. Lege einen Index auf note an und wiederhole ein ähnliches Update.

<details>
<summary>Show solution</summary>
<p>

Lösung/Erklärung:

- Ohne Index auf note können Updates, die nur note ändern, häufig HOT sein (kein Index-Update nötig).
- Mit Index auf note muss PostgreSQL bei jeder Änderung an note auch den Index aktualisieren ⇒ HOT wird verhindert.

Index anlegen:

```sql
CREATE INDEX idx_demo_hot_note
ON demo_hot(note);
```

Merksatz:

Viele/„falsche“ Indexe machen Updates deutlich teurer.

</p>
</details>

## 10.8 Reindexing & Wartung (Bloat/Statistiken)

**Aufgabe:**

Finde Indexe, die wenig genutzt werden, und führe eine Wartungsmaßnahme durch.

1. Welche Indexe werden selten genutzt?

```sql
SELECT
  relname AS table_name,
  indexrelname AS index_name,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

2. Wähle einen Index aus, der potenziell „problematisch“ ist (sehr niedrige Nutzung) und diskutiere:

- Warum könnte er existieren?
- Warum wird er nicht genutzt?
- Ist er überflüssig oder nur selten gebraucht?

3. Führe testweise (wenn sinnvoll) REINDEX aus:

```sql
REINDEX INDEX <index_name>;
```

Optional: VACUUM (ANALYZE) auf einer betroffenen Tabelle.

<details>
<summary>Show solution</summary>
<p>

Lösungshinweise:

- Niedriges idx_scan kann „ungenutzt“ bedeuten – oder „selten, aber wichtig“ (z. B. Monatsreport).
- REINDEX kann sinnvoll sein bei:
  - Index-Bloat
  - Korruption (selten)
  - stark veränderten Datenmustern (eher Stats/ANALYZE zuerst)

Standard-Wartung (häufig sinnvoller als REINDEX):

```sql
VACUUM (ANALYZE) orders;
VACUUM (ANALYZE) order_details;
```

</p>
</details>
