- [8. Fehler- & Ausnahmebehandlung](#8-fehler---ausnahmebehandlung)
  - [8.1 RAISE NOTICE/WARNING zur Laufzeitdiagnose](#81-raise-noticewarning-zur-laufzeitdiagnose)
  - [8.2 Fachliche Validierung mit RAISE EXCEPTION (inkl. ERRCODE)](#82-fachliche-validierung-mit-raise-exception-inkl-errcode)
  - [8.3 Exception Handling mit WHEN (unique_violation)](#83-exception-handling-mit-when-unique_violation)

Bearbeitungszeit: 45 Minuten

# 8. Fehler- & Ausnahmebehandlung

In dieser Aufgabe übst du den praktischen Umgang mit Fehlern und Ausnahmen in PL/pgSQL. Du nutzt dabei typische Problemfälle aus der Northwind-Datenbank: doppelte Schlüssel, fehlende Fremdschlüssel, ungültige Eingaben und „teilweise erfolgreiche“ Batch-Operationen.

## 8.1 RAISE NOTICE/WARNING zur Laufzeitdiagnose

Erstelle eine Funktion `log_order_info(p_order_id int)`, die:

- prüft, ob eine Bestellung mit `order_id = p_order_id` existiert
- wenn nicht vorhanden: `RAISE WARNING` mit einer sinnvollen Meldung und RETURN (kein Exception-Abbruch)
- wenn vorhanden: `RAISE NOTICE` mit:
  - `order_id`
  - `customer_id`
  - `order_date`

Führen Sie danach folgende Methoden aus

```sql
CALL log_order_info(10248);
CALL log_order_info(-999);
```

**Erwartetes Ergebnis**

```text
NOTICE:  Order 10248 | Customer: VINET | Date: 1996-07-04
WARNING:  Order -999 existiert nicht.
```

<details>
<summary>Show solution</summary>
<p>

**/**

```sql
CREATE OR REPLACE PROCEDURE log_order_info(p_order_id int)
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer_id text;
    v_order_date date;
BEGIN
    SELECT o.customer_id, o.order_date
    INTO v_customer_id, v_order_date
    FROM orders o
    WHERE o.order_id = p_order_id;

    IF NOT FOUND THEN
        RAISE WARNING 'Order % existiert nicht.', p_order_id;
        RETURN;
    END IF;

    RAISE NOTICE 'Order % | Customer: % | Date: %', p_order_id, v_customer_id, v_order_date;
END;
$$;

-- Test
SELECT log_order_info(10248);
SELECT log_order_info(-999);
```

</p>
</details>

## 8.2 Fachliche Validierung mit RAISE EXCEPTION (inkl. ERRCODE)

Erstelle eine Funktion `assert_positive_freight(p_order_id smallint)`, die:

- den `freight`-Wert der Bestellung liest
- wenn `order_id` nicht existiert → `RAISE EXCEPTION` mit **eigenem ERRCODE** (z. B. `P0001`)
- wenn `freight <= 0` → `RAISE EXCEPTION` mit **eigenem ERRCODE** (z. B. `P0002`)
- sonst: RETURN `freight`

Führen Sie danach folgende Methoden aus

```sql
SELECT assert_positive_freight(1::smallint);
SELECT assert_positive_freight(10248::smallint);

```

<details>
<summary>Show solution</summary>
<p>

```sql
CREATE OR REPLACE FUNCTION assert_positive_freight(p_order_id smallint)
RETURNS numeric
LANGUAGE plpgsql
AS $$
DECLARE
    v_freight numeric;
BEGIN
    SELECT o.freight
    INTO v_freight
    FROM orders o
    WHERE o.order_id = p_order_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION
            USING MESSAGE = format('Order % existiert nicht.', p_order_id),
                  ERRCODE = 'P0001';
    END IF;

    IF v_freight IS NULL OR v_freight <= 0 THEN
        RAISE EXCEPTION
            USING MESSAGE = format('Freight muss > 0 sein. Order: %, Freight: %', p_order_id, v_freight),
                  ERRCODE = 'P0002';
    END IF;

    RETURN v_freight;
END;
$$;

-- Test
SELECT assert_positive_freight(1);
```

</p>
</details>

## 8.3 Exception Handling mit WHEN (unique_violation)

Erstelle eine Tabelle order_audit zur Protokollierung von Bestellungen:

- `order_id smallint PRIMARY KEY`
- `logged_at timestamptz NOT NULL DEFAULT now()`

Erstelle danach eine Funktion `audit_order(p_order_id smallint)`, die:

- einen Datensatz in `order_audit` einfügt
bei `unique_violation` (doppelte order_id) **nicht abbrechen**, sondern `RAISE NOTICE` „bereits vorhanden“

<details>
<summary>Show solution</summary>
<p>

**/**

```sql
CREATE TABLE IF NOT EXISTS order_audit (
    order_id smallint PRIMARY KEY,
    logged_at timestamptz NOT NULL DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_order(p_order_id smallint)
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO order_audit(order_id)
    VALUES (p_order_id);

    RAISE NOTICE 'Audit-Eintrag erstellt für Order %', p_order_id;

EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Audit-Eintrag existiert bereits für Order %', p_order_id;
END;
$$;

-- Test
SELECT audit_order(1);
SELECT audit_order(1);
```

</p>
</details>

## 8.4 Batch-Verarbeitung mit „Teil-Erfolg“ (implizite Savepoints)

Erstelle eine Funktion `audit_recent_orders(p_limit int)`, die:

- die neuesten `p_limit` Bestellungen aus `orders` iteriert (z. B. nach `order_date DESC NULLS LAST`, dann `order_id DESC`)
- für jede Bestellung `audit_order(order_id)` aufruft
- **Fehler pro Datensatz** abfängt (`WHEN others`) und als `RAISE WARNING` protokolliert
- am Ende die Anzahl erfolgreicher Inserts zurückgibt

Hinweis: Durch `BEGIN … EXCEPTION … END` entsteht ein impliziter Savepoint pro Iteration.

<details>
<summary>Show solution</summary>
<p>

**/**

```sql
CREATE OR REPLACE FUNCTION audit_recent_orders(p_limit int)
RETURNS int
LANGUAGE plpgsql
AS $$
DECLARE
    rec record;
    v_ok int := 0;
BEGIN
    FOR rec IN
        SELECT o.order_id
        FROM orders o
        ORDER BY o.order_date DESC NULLS LAST, o.order_id DESC
        LIMIT p_limit
    LOOP
        BEGIN
            PERFORM audit_order(rec.order_id);
            v_ok := v_ok + 1;
        EXCEPTION
            WHEN others THEN
                RAISE WARNING 'Fehler bei Order %: %', rec.order_id, SQLERRM;
                -- weiter mit nächstem Datensatz
        END;
    END LOOP;

    RETURN v_ok;
END;
$$;

-- Test
SELECT audit_recent_orders(20);
```

</p>
</details>
