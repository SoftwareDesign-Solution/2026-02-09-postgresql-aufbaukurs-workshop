- [7. Trigger und Triggerfunktionen](#7-trigger-und-triggerfunktionen)
  - [7.1 Validierung mit BEFORE Trigger](#71-validierung-mit-before-trigger)
  - [7.2 Audit-Log mit AFTER Trigger](#72-audit-log-mit-after-trigger)
  - [7.3 Analyse mit TG_OP und Trigger-Parametern](#73-analyse-mit-tg_op-und-trigger-parametern)

Bearbeitungszeit: 45 Minuten

# 7. Trigger und Triggerfunktionen

In der PostgreSQL Northwind Database sollen Trigger eingesetzt werden, um
Geschäftsregeln und Audit-Anforderungen direkt auf Datenbankebene umzusetzen.

Du arbeitest mit folgenden Tabellen:

- `orders`
- `order_details`

## 7.1 Validierung mit BEFORE Trigger

**Ziel:**

In der Tabelle `order_details` darf keine negative Menge (`quantity`) gespeichert werden.

**Anforderungen**

- Erstelle eine **Triggerfunktion**
- Erstelle einen **BEFORE INSERT OR UPDATE Trigger**
- Wenn `quantity < 0`, soll die Operation mit einer Fehlermeldung abgebrochen werden
- Nutze `NEW`

<details>
<summary>Show solution</summary>
<p>

**Triggerfunktion**

```sql
CREATE OR REPLACE FUNCTION validate_order_quantity()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.quantity < 0 THEN
    RAISE EXCEPTION 'Quantity darf nicht negativ sein!';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Trigger**

```sql
CREATE TRIGGER order_details_quantity_check
BEFORE INSERT OR UPDATE ON order_details
FOR EACH ROW
EXECUTE FUNCTION validate_order_quantity();
```

</p>
</details>

## 7.2 Audit-Log mit AFTER Trigger

**Ziel:**

Änderungen an Bestellpositionen sollen protokolliert werden.

**Anforderungen**

- Erstelle eine Tabelle `order_details_audit`
- Protokolliert werden:
  - `order_id`
  - `product_id`
  - alte & neue `quantity`
  - Art der Operation (`INSERT` oder `UPDATE`)
  - Zeitstempel
- Nutze:
  - `OLD`
  - `NEW`
  - `TG_OP`
- Trigger soll **AFTER INSERT OR UPDATE** ausgeführt werden

<details>
<summary>Show solution</summary>
<p>

**Audit-Tabelle**

```sql
CREATE TABLE order_details_audit (
  id SERIAL PRIMARY KEY,
  order_id INT,
  product_id INT,
  old_quantity INT,
  new_quantity INT,
  operation TEXT,
  changed_at TIMESTAMP DEFAULT now()
);
```

**Triggerfunktion**

```sql
CREATE OR REPLACE FUNCTION audit_order_details()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO order_details_audit (
      order_id, product_id, new_quantity, operation
    ) VALUES (
      NEW.order_id, NEW.product_id, NEW.quantity, 'INSERT'
    );
  END IF;

  IF TG_OP = 'UPDATE' THEN
    INSERT INTO order_details_audit (
      order_id, product_id, old_quantity, new_quantity, operation
    ) VALUES (
      OLD.order_id,
      OLD.product_id,
      OLD.quantity,
      NEW.quantity,
      'UPDATE'
    );
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Trigger**

```sql
CREATE TRIGGER order_details_audit_trigger
AFTER INSERT OR UPDATE ON order_details
FOR EACH ROW
EXECUTE FUNCTION audit_order_details();
```

</p>
</details>

## 7.3 Analyse mit TG_OP und Trigger-Parametern

**Ziel:**

Analysiere das Verhalten des Triggers anhand von Testdaten.

**Aufgaben**

1. Führe ein `INSERT` in order_details aus
2. Führe ein `UPDATE` der quantity aus
3. Prüfe die Inhalte der Tabelle `order_details_audit`

**Beantworte folgende Fragen:**

- Welche Werte hat `TG_OP` bei INSERT / UPDATE?
- Wann ist `OLD` verfügbar?
- Wann ist `NEW` verfügbar?
- Warum ist ein **AFTER Trigger** für Audit-Logs sinnvoller als ein **BEFORE Trigger**?

<details>
<summary>Show solution</summary>
<p>

**TG_OP**

- INSERT → `'INSERT'`
- UPDATE → `'UPDATE'`

**OLD / NEW**

| Operation | OLD | NEW |
| --------- | --- | --- |
| INSERT | ❌ | ✅ |
| UPDATE | ✅ | ✅ |

**Warum AFTER Trigger?**

- Audit soll nur erfolgreiche Änderungen loggen
- Bei BEFORE Triggern könnte die Operation noch abbrechen
- AFTER Trigger sehen den finalen Zustand der Daten

</p>
</details>
