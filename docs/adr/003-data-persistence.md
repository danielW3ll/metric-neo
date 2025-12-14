 # ADR 003: Datenhaltung, Snapshot-Strategie und JSON-Struktur

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
Wir trennen die Daten physisch auf dem Dateisystem nach fachlichen Aggregaten:

#### Stammdaten (Mutable Master Data)
*   `/data/inventory/profiles/`: Sportgeräte-Profile (Waffen/Bögen). Dateinamen: `{UUID}.json`
*   `/data/inventory/projectiles/`: Munitions-Stammdaten (Diabolos, Pfeile). Dateinamen: `{UUID}.json`
*   `/data/inventory/sights/`: Zieloptiken (Scopes, RedDots) als eigenständige Entitäten. Dateinamen: `{UUID}.json`
    *   **Hinweis:** Profile referenzieren Sights via UUID. Die Sight-Daten werden beim Session-Erstellen als Teil des Profile-Snapshots eingefroren.

#### Transaktionsdaten (Immutable Transaction Data)
*   `/data/sessions/`: Unveränderliche Messprotokolle. Jede Session erhält eine eigene JSON-Datei (z.B. `{UUID}.json`).
    *   Enthält vollständige Snapshots von Profile + Projectile + Sight (falls vorhanden).

### 3. Das Snapshot-Pattern (Deep Copy)
Um die historische Integrität zu garantieren, implementieren wir beim Erstellen einer Session einen **Deep Copy Mechanismus**:
*   Startet der Nutzer eine Session, werden die **aktuellen Werte** aus drei Quellen vollständig kopiert:
    1. **Profile** (aus `/data/inventory/profiles/{UUID}.json`)
    2. **Projectile** (aus `/data/inventory/projectiles/{UUID}.json`)
    3. **Sight** (falls im Profile referenziert, aus `/data/inventory/sights/{UUID}.json`) – wird als Teil des Profile-Snapshots eingebettet
*   Die Session referenziert *nicht* die Live-Daten via UUIDs, sondern besitzt ihren eigenen, eingefrorenen Datenstand ("Snapshot").
*   Spätere Änderungen im Inventar (z.B. Nutzer korrigiert Projektilgewicht, tauscht Optik) haben somit **keinen Einfluss** auf existierende Sessions.

## Technische Umsetzung (Schema-Beispiele)

### Session-Datei (`/data/sessions/{UUID}.json`)
Enthält alle relevanten Daten für die Audit-Trail-Anforderung:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2023-12-01T10:00:00Z",
  "note": "Training vor Wettkampf",
  "temperature_celsius": 21.5,
  
  // EINGEFRORENER SNAPSHOT: Waffe + eingebettete Optik
  "profile_snapshot": {
    "original_id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "Steyr Challenge E",
    "category": "AirRifle",
    "barrel_length_mm": 450,
    "trigger_weight_g": 500,
    "sight_height_mm": 45,
    
    // Eingebettete Optik (Deep Copy aus /data/inventory/sights/)
    "sighting_system": {
      "type": "Scope",
      "model_name": "Schmidt & Bender PM II 12x50",
      "weight_g": 850,
      "min_magnification": 3,
      "max_magnification": 12
    }
  },
  
  // EINGEFRORENER SNAPSHOT: Munition
  "projectile_snapshot": {
    "original_id": "789e0123-e45b-67c8-d901-234567890abc",
    "name": "JSB Exact Heavy",
    "charge": "LOT-2023-11-A",
    "weight_g": 0.547,
    "bc": 0.022
  },

  // MESSWERTE (Beziehen sich ausschließlich auf Snapshots)
  "shots": [
    { "timestamp": "2023-12-01T10:05:23Z", "velocity_mps": 175.5, "valid": true },
    { "timestamp": "2023-12-01T10:06:01Z", "velocity_mps": 176.2, "valid": true }
  ]
}
```

### Profile-Datei (`/data/inventory/profiles/{UUID}.json`)
Referenziert Sight via UUID (wird beim Session-Erstellen aufgelöst):

```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Steyr Challenge E",
  "category": "AirRifle",
  "barrel_length_mm": 450,
  "trigger_weight_g": 500,
  "sight_id": "999e8888-e77b-66c6-d555-333344445555",
  "default_projectile_id": "789e0123-e45b-67c8-d901-234567890abc"
}
```

### Projectile-Datei (`/data/inventory/projectiles/{UUID}.json`)
Munitions-Stammdaten mit Chargen-Tracking:

```json
{
  "id": "789e0123-e45b-67c8-d901-234567890abc",
  "name": "JSB Exact Heavy",
  "charge": "LOT-2023-11-A",
  "weight_g": 0.547,
  "bc": 0.022
}
```

### Sight-Datei (`/data/inventory/sights/{UUID}.json`)
Eigenständige Entität für Wiederverwendbarkeit:

```json
{
  "id": "999e8888-e77b-66c6-d555-333344445555",
  "type": "Scope",
  "model_name": "Schmidt & Bender PM II 12x50",
  "weight_g": 850,
  "min_magnification": 3,
  "max_magnification": 12
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