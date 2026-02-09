- [1. PostgreSQL Northwind-Datenbank einrichten (Download & Import)](#1-postgresql-northwind-datenbank-einrichten-download--import)
  - [1.1 Repository öffnen, Northwind-Dateien finden und importieren](#11-repository-öffnen-northwind-dateien-finden-und-importieren)

Bearbeitungszeit: 30 Minuten

# 1. PostgreSQL Northwind-Datenbank einrichten (Download & Import)

In dieser Aufgabe richtest du die Übungsdatenbank PostgreSQL Northwind für den Kurs ein.
Die Datenbank liegt im Schulungsrepository im Verzeichnis database/. Du sollst sie lokal importieren, sodass du danach per SQL darauf zugreifen kannst.

Ziel: Am Ende kannst du dich mit der Datenbank verbinden und eine erste Abfrage (z. B. Tabellen anzeigen) ausführen.

## 1.1 Repository öffnen, Northwind-Dateien finden und importieren

1. Repository öffnen / aktualisieren
    - Stelle sicher, dass du das Schulungsrepository lokal hast (z. B. git pull).
    - Navigiere in das Verzeichnis database/.

2. Northwind-Datei identifizieren
    - Suche im Ordner nach einer Import-Datei, typischerweise z. B.:
        - northwind.sql (SQL-Dump) oder
        - northwind.tar / northwind.backup (pg_dump Custom Format) oder
        - .psql / .dump
    - Merke dir den Dateinamen.
3. Datenbank in PostgreSQL anlegen
    - Lege eine neue Datenbank an, z. B. northwind.
4. Import durchführen
    - Importiere die Datei entweder:
    - mit psql (bei .sql) oder
    - mit pg_restore (bei .backup/.tar) oder
    - über einen DB-Client (z. B. DBeaver, DataGrip, pgAdmin).
5. Import prüfen
    - Verbinde dich zur Datenbank und prüfe, ob Tabellen vorhanden sind (z. B. \dt oder eine Query auf information_schema.tables).

<details>
<summary>Show solution</summary>
<p>

**Beispiel-Lösung (psql / CLI)**

> **Wichtig**: Passe Benutzer/Host/Port/Dateiname an deine Umgebung an.

**A) Wenn du eine northwind.sql hast (Plain SQL)**

```bash
# 1) DB anlegen
createdb -U postgres northwind

# 2) Import
psql -U postgres -d northwind -f database/northwind.sql

# 3) Prüfen
psql -U postgres -d northwind -c "\dt"
```

**B) Wenn du ein Dump-Format wie .backup oder .tar hast**

```bash
# 1) DB anlegen
createdb -U postgres northwind

# 2) Import via pg_restore
pg_restore -U postgres -d northwind database/northwind.backup

# 3) Prüfen
psql -U postgres -d northwind -c "\dt"
```

**Beispiel-Lösung (SQL zum Anlegen + Prüfen im Client)**

**1) Datenbank anlegen**

```sql
CREATE DATABASE northwind;
```

**2) Import im DB-Client**
    - Öffne dein Tool (z. B. pgAdmin/DBeaver/DataGrip)
    - Verbindung zu PostgreSQL herstellen
    - Datenbank northwind auswählen/verbinden
    - Import ausführen:
        - bei .sql: „Run SQL Script“ / „Execute Script“
        - bei .backup/.tar: Restore-Funktion verwenden

**3) Prüfen, ob Tabellen da sind**
    ```sql
    SELECT table_schema, table_name
    FROM information_schema.tables
    WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
    ORDER BY table_schema, table_name;
    ```

**Typische Fehler & Fixes**

- Permission denied / role does not exist
    → richtigen User verwenden (-U ...) oder Rolle anlegen.

- Falsche DB ausgewählt
    → prüfen, ob du wirklich in northwind verbunden bist.

- Encoding/Locale-Probleme
    → DB mit UTF-8 anlegen (Standard) oder Dump-Quelle prüfen.

</p>
</details>
