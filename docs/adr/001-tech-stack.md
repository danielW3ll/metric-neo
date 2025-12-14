# ADR 001: Technologie-Stack und Plattform-Strategie

* **Status:** Offene Entscheidung
* **Datum:** 2025-12-09
* **Autor:** Daniel Wellermann
* **Projekt:** Metric Neo

## Kontext und Problemstellung

### Fachliche Motivation (Business Case)
Aktuell existiert am Markt keine herstellerübergreifende Software-Lösung, die ballistische Messdaten (Chronograph) mit einer umfassenden Inventar-Verwaltung verknüpft.
Die vom Hardware-Hersteller (LMBR) bereitgestellten Tools beschränken sich auf die reine Anzeige von Momentanwerten.

Für Anwender im professionellen und ambitionierten Sportbereich ergeben sich dadurch folgende Probleme:
1.  **Mangelnde Übersicht (Skalierbarkeit):** Sobald ein Schütze mehr als drei Sportgeräte (z.B. verschiedene Bogen-Setups oder Gewehre) verwaltet, wird die manuelle Protokollierung (Papier/Excel) unübersichtlich und fehleranfällig.
2.  **Fehlende Vergleichbarkeit:** Es fehlt die Möglichkeit, historische Messreihen verschiedener Geräte oder Munitionstypen (Projektile) visuell übereinanderzulegen ("Overlay"), um Leistungsveränderungen oder Wartungsbedarf zu erkennen.
3.  **Wartungs-Lücke:** Es gibt keine digitale Verknüpfung zwischen Schussleistung (Energieverlust) und Wartungsintervallen (z.B. Federwechsel, Sehnenverschleiß).

## Kontext und Problemstellung
Für das Projekt "Metric Neo" wird eine moderne Desktop-Anwendung benötigt, die ballistische Messdaten von externer Hardware (LMBR Chronograph) via RS232 ausliest (im ersten Schritt), mit einem umfassenden **Inventar-Verwaltungssystem** verknüpft und historisch revisionssicher archiviert.

Die Anwendung muss folgende Kernfunktionen abbilden:
1. **Geräte-Inventarisierung:** Verwaltung mehrerer Sportgeräte (Profile) mit konfigurierbaren Eigenschaften (Lauflänge, Abzugsgewicht, montierte Zieloptiken).
2. **Munitions-Datenbank:** Zentrale Verwaltung verschiedener Projektile (Diabolo, Pfeile) mit physikalischen Eigenschaften (Gewicht, ballistischer Koeffizient).
3. **Session-basierte Messungen:** Erfassung zusammenhängender Messreihen mit automatischem Snapshot der verwendeten Konfiguration (Waffe + Munition + Umgebungsbedingungen) zum Zeitpunkt der Messung.
4. **Audit-Trail & Historie:** Unveränderliche Archivierung historischer Daten durch Deep-Copy-Mechanismus – Änderungen an Stammdaten dürfen nachträglich berechnete Energiewerte alter Sessions nicht beeinflussen.
5. **Wartungs-Tracking:** Langfristige Analyse von Leistungsdegradation (Geschwindigkeitsverlust) zur Ableitung von Wartungsintervallen (z.B. Federwechsel, Sehnenverschleiß).

Die Anwendung muss sowohl auf **Windows** (breite Masse im Sportbereich) als auch auf **Linux** (technische Nutzer, Sicherheitsfokus) lauffähig sein.
Da das Projekt als Referenz dient und später potenziell erweitert wird, muss die Architektur eine Balance zwischen **Entwicklungsgeschwindigkeit** (Time-to-Market), **Wartbarkeit** und **Betriebssicherheit** finden.

## Entscheidung

Wir entscheiden uns für folgende Kern-Komponenten:
1.  **Framework:** Wails (Go Backend + Web Frontend).
2.  **UI Library:** Vue.js mit Element Plus.
3.  **Datenhaltung:** JSON-Dateien (File System Storage).
4.  **Lizenz-Management:** Pragmatischer "Permissive First" Ansatz.

### Begründung
*   **Wails (Go):** Go bietet als kompilierte Sprache eine hohe Ausführungssicherheit (Typensicherheit) und exzellente Performance für die serielle Kommunikation (RS232). Im Gegensatz zu Electron erzeugt Wails deutlich schlankere Binaries und verbraucht weniger RAM, was auf älteren Laptops am Schießstand wichtig ist.
*   **Vue.js & Element Plus:** Element Plus bietet einen sehr umfangreichen Satz an getesteten UI-Komponenten (Data Grids, Inputs). Da diese unter der MIT-Lizenz stehen, sind sie unproblematisch. Dies beschleunigt die UI-Entwicklung massiv im Vergleich zu nativem C++/GTK Code.
*   **JSON-Storage:** In der Startphase ist eine SQL-Datenbank (auch SQLite) oft "Over-Engineering". JSON ist menschenlesbar. Wenn ein User meldet "Meine Waffe wird falsch geladen", kann er uns einfach die JSON-Datei schicken. Das erleichtert Support und Debugging drastisch.
*   **Plattform-Strategie:** Wir schließen Windows nicht aus, da dort 90% der Zielgruppe (Schützenvereine) unterwegs sind. Linux wird als First-Class-Citizen unterstützt, um technisch versierten Nutzern eine auditierbare Plattform zu bieten.

## Konsequenzen

### Positiv
*   **Single Codebase:** Wir warten nur einen Code-Strang für Windows und Linux.
*   **Security by Design:** Go schützt vor vielen klassischen C-Fehlern (Buffer Overflows), was die Stabilität der RS232-Verarbeitung erhöht.
*   **Transparenz:** Durch JSON als Speicherformat hat der Nutzer volle Kontrolle und Einsicht in seine Daten.

### Negativ
*   **Build-Komplexität:** Das Kompilieren für verschiedene Plattformen (Cross-Compile) erfordert ein sauber eingerichtetes Build-System (Docker oder CI-Pipeline).
*   **Lizenz-Prüfung:** Wir müssen bei jedem neuen NPM-Paket kurz prüfen, ob die Lizenz passt (kein striktes Copyleft/GPL, das unseren Code "infizieren" würde), um uns kommerzielle Optionen offen zu halten.