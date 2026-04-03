# Dual Decay: Nutzung schlaegt Alter

## Problem

Temporal Decay in der ersten Version basiert ausschliesslich auf `file_mtime` (Zeitpunkt der letzten Datei-Aenderung):

```
decay = exp(-lambda * tage_seit_aenderung)
```

Halbwertszeit: 90 Tage. Nach 6 Monaten hat ein Dokument nur noch ~16% seines urspruenglichen Scores. Nach 12 Monaten: ~2.5%.

**Das Problem:** Ein Projekt, an dem seit 3 Monaten niemand gearbeitet hat, verschwindet aus den Suchergebnissen. Obwohl es letzte Woche 5x abgerufen wurde.

Realer Fall: Das Echterdingen-Erbschaftsprojekt. Monate ohne Datei-Aenderung, aber regelmaessig nachgeschlagen fuer Aktenzeichen, Fristen, Kontakte.

## Loesung: Zwei Decay-Dimensionen

```
decay_time  = exp(-lambda_1 * tage_seit_datei_aenderung)
decay_usage = exp(-lambda_2 * tage_seit_letztem_retrieval)
final_decay = max(decay_time, decay_usage)
```

### Warum `max()` statt Durchschnitt?

- Wenn du **gestern danach gesucht** hast, ist es relevant. Egal wie alt die Datei ist.
- Wenn die **Datei gestern geaendert** wurde, ist sie relevant. Egal ob du nie danach gesucht hast.

Der bessere Wert gewinnt. `max()` ist die mathematisch korrekte Abbildung von "relevant, wenn aus irgendeinem Grund aktuell".

Ein Durchschnitt wuerde ein gestern genutztes aber 6 Monate altes Dokument abwerten. Das ist falsch.

## Implementation

### Datenmodell

Neue Spalte in der `chunks`-Tabelle (oder separate Tracking-Tabelle):

```sql
ALTER TABLE chunks ADD COLUMN last_accessed TEXT;  -- ISO 8601 Timestamp
```

### Scoring-Pipeline

Bei jedem Search-Hit:
1. Ergebnis-Chunks zurueckgeben (wie bisher)
2. `last_accessed` fuer alle getroffenen Chunks aktualisieren

Bei jeder neuen Suche:
1. `decay_time` aus `file_mtime` berechnen (wie bisher)
2. `decay_usage` aus `last_accessed` berechnen
3. `final_decay = max(decay_time, decay_usage)`
4. `final_score *= final_decay`

### Feature-Flag

```python
use_dual_decay: bool = False  # Aktivierung ueber RunConfig
usage_decay_half_life_days: int = 60  # Eigene Halbwertszeit
```

## Evergreen-Ausnahme

Dokumente die als `evergreen` markiert sind (MEMORY.md, CLAUDE.md, README.md) sind weiterhin von jeglichem Decay ausgenommen. Dual Decay betrifft nur regulaere Dokumente.

## Testbarkeit

Im Synthetic Test Harness:
- Synthetische Dokumente mit altem `file_mtime` aber kuerzlichem `last_accessed`
- Vergleich: reiner Time-Decay vs. Dual-Decay
- Erwartung: Dual-Decay findet das kuerzlich abgerufene Dokument, Time-Decay nicht

## Status

Konzept definiert (April 2026). Implementation als Teil des Synthetic Test Harness geplant.
