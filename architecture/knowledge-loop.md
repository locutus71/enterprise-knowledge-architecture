# Knowledge Loop

## Kernidee

Das System wird mit jeder Session besser. Nicht weil jemand es trainiert, sondern weil die Architektur Lernen erzwingt. Kein Fine-Tuning, kein manuelles Pflegen. Es lernt, weil mit ihm gearbeitet wird.

## Der Zyklus (5 Schritte)

```
1. Session → 2. Destillation → 3. Storage → 4. Export → 5. Reindex
     ↑                                                        ↓
     ←←←←←←←← Naechste Session (besserer Kontext) ←←←←←←←←←←
```

### Schritt 1: Session

Normale Arbeitssitzung mit dem KI-Agenten. Recherchen, Analysen, Dokument-Erstellung, Debugging. Jede Session produziert implizites Wissen.

### Schritt 2: Destillation

Am Ende jeder Session (getriggert durch Stop-Hook):
- Was wurde gelernt?
- Welche Fehler wurden gemacht?
- Welche Entscheidungen getroffen?
- Welche nicht-offensichtlichen Loesungen gefunden?

Das Ergebnis sind strukturierte Notes und Logs in Deep Storage. Nicht der gesamte Chat-Verlauf, sondern das destillierte Wissen daraus.

### Schritt 3: Storage

Notes und Logs werden in Deep Storage (Layer 2) geschrieben mit:
- Titel und Inhalt
- Tags fuer Kategorisierung
- Projekt-Zuordnung
- Confidence Score (verified/unverified)
- Automatischer Timestamp

### Schritt 4: Export

Ein Export-Script wandelt Deep Storage Notes in Markdown-Dateien um.
- Nur Notes mit `confidence != rejected` werden exportiert
- Markdown-Format fuer die Search Engine optimiert
- Metadaten (Autor, Datum, Tags) als Frontmatter

### Schritt 5: Reindex

Die Search Engine (Layer 3) reindexiert alle Dokumente inklusive der neuen Exporte.
- Neue Chunks werden erzeugt
- Embeddings berechnet
- BM25-Index aktualisiert
- Temporal Decay Scores neu berechnet

Schwellenwerte fuer automatischen Reindex:
- 500k Token verbraucht ODER
- 100 Suchanfragen seit letztem Reindex ODER
- 5+ geaenderte Dateien

## Warum es funktioniert

Weil es nicht optional ist. Hooks erzwingen jeden Schritt:
- **Stop-Hook** triggert Destillation und Export
- **PreCompact-Hook** sichert Zwischenergebnisse bei langen Sessions
- **SessionStart-Hook** laedt die Ergebnisse der vorherigen Session

Der Knowledge Loop ist kein Feature, das man aktiviert. Er ist eine Konsequenz der Hook-Architektur.

## Compound Effect

Session 1 produziert Wissen A.
Session 2 hat A im Kontext und produziert Wissen B (informiert durch A).
Session 10 hat A-I im Kontext und kann Zusammenhaenge erkennen die in keiner einzelnen Session existierten.

Das ist der eigentliche Durchbruch: Nicht dass das System Fakten speichert, sondern dass es ueber Zeit Zusammenhaenge aufbaut die kein Mensch manuell pflegen koennte.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
