 ADR 003: Datenhaltung, Snapshot-Strategie und JSON-Struktur

* **Status:** Offene Entscheidung
* **Datum:** 2025-12-11
* **Autor:** Daniel Wellermann
* **Projekt:** Metric Neo

## Kontext und Problemstellung
Das System muss "High Assurance" Kriterien erfüllen. Eine zentrale Anforderung ist die **Revisionssicherheit (Audit Trail)** historischer Messdaten.
Ein klassisches Problem in Datenbankanwendungen ist "Mutable State": Wenn der Nutzer die Stammdaten eines Projektils (z.B. Gewicht) ändert, darf sich die berechnete Energie alter Schussprotokolle **nicht** verändern.

Zudem muss die Datenhaltung:
1.  **Lokal & Offline** funktionieren (File-based).
2.  **Backup-freundlich** sein (einfaches Kopieren).
3.  Das komplexe, verschachtelte Domain Model (Session -> Profile -> SightingSystem) effizient abbilden.

## Entscheidung

### 1. Speicherformat: JSON Document Store
Wir verwenden **JSON-Dateien** als primären Datenspeicher.
Das Domain Model ist hierarchisch (Tree-Structure). JSON bildet dies nativ ab, ohne komplexe ORM-Mapper (Object-Relational Mapping) oder Joins, die bei SQL notwendig wären.

### 2. Trennung: Master vs. Transaction Data
Wir trennen die Daten physisch auf dem Dateisystem:
*   `/data/inventory/`: Veränderliche Stammdaten (Profile, Projektile).
*   `/data/sessions/`: Unveränderliche Messprotokolle. Jede Session erhält eine eigene JSON-Datei (z.B. `{UUID}.json`).

### 3. Das Snapshot-Pattern (Deep Copy)
Um die historische Integrität zu garantieren, implementieren wir beim Erstellen einer Session einen **Deep Copy Mechanismus**:
*   Startet der Nutzer eine Session, werden die **aktuellen Werte** aus dem Inventar (Waffe, Optik, Munition) vollständig in die Session-Datei kopiert.
*   Die Session referenziert *nicht* die Live-Daten, sondern besitzt ihren eigenen, eingefrorenen Datenstand ("Snapshot").
*   Spätere Änderungen im Inventar haben somit **keinen Einfluss** auf existierende Sessions.

## Technische Umsetzung (Schema-Beispiel)

Die Dateistruktur einer Session-Datei (`/data/sessions/xyz.json`) muss das vollständige Profil beinhalten:

```json
{
  "id": "uuid-v4",
  "created_at": "2023-12-01T10:00:00Z",
  
  // EINGEFRORENER SNAPSHOT (Deep Copy)
  "profile_snapshot": {
    "name": "Steyr Challenge",
    "trigger_weight_g": 500,
    "sighting_system": {
        "type": "Scope",
        "weight_g": 850
    }
  },
  
  // EINGEFRORENER SNAPSHOT (Deep Copy)
  "projectile_snapshot": {
    "name": "JSB Exact Charge 1",
    "weight_g": 0.547
  },

  // MESSWERTE (Beziehen sich auf den Snapshot)
  "shots": [
    { "ts": "...", "v_mps": 175.5, "valid": true }
  ]
}
```

## Konsequenzen

### Positiv
Audit-Sicherheit: Historische Daten sind physikalisch von aktuellen Stammdaten entkoppelt. Eine Manipulation ist durch die redundante Speicherung ausgeschlossen (außer durch manuelles Editieren der Datei).

Performance: Lazy Loading ist trivial. Für eine Listenansicht müssen nur kleine Metadaten geladen werden; die großen Session-Dateien werden nur bei Bedarf ("On Click") geladen.

Debugging: Fehlerhafte Berechnungen können durch Zusenden der einen JSON-Datei reproduziert werden, da sie alle relevanten Parameter enthält.

### Negativ

Redundanz: Der Speicherbedarf ist höher als bei normalisierten SQL-Tabellen, da Waffendaten in jeder Session dupliziert werden. Da es sich um Textdaten im Kilobyte-Bereich handelt, ist dies bei heutiger Hardware vernachlässigbar.

Refactoring: Ändert sich die Struktur des Domain Models drastisch, müssen Migrations-Skripte geschrieben werden, die alte JSON-Dateien beim Einlesen aktualisieren.