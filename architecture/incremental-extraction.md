# Incremental Knowledge Extraction

## Problem

LLMs haben ein begrenztes Context-Fenster. Bei langen Sessions wird der Kontext komprimiert (Autocompact). Alles was bis dahin nicht explizit gespeichert wurde, geht verloren.

Ein Session-Ende-Hook hilft nicht, wenn die Session vorher 2x komprimiert wird. Die Erkenntnisse aus den ersten 70% der Konversation sind dann bereits weg.

## Loesung: Eventbasierte Extraktion

Statt auf das Session-Ende zu warten, feuert die Extraktion bei drei Triggern:

| Trigger | Wann | Was passiert |
|---------|------|-------------|
| **PreCompact-Hook** | Automatisch vor jeder Context-Komprimierung | Extrahiert Fakten, Relationen, Projektstand aus dem noch vollstaendigen Kontext |
| **Intervall-basiert** | Alle ~15 Tool-Aufrufe oder bei Meilensteinen | Agent speichert proaktiv als Teil seiner Arbeitsroutine |
| **Session-Ende** | User beendet die Session | Nur noch Nachzuegler seit dem letzten Trigger |

## Implementation

### PreCompact-Hook (Sicherheitsnetz)

Der Hook ist in der Claude Code `settings.json` als `PreCompact`-Event registriert und fuehrt zwei Aktionen aus:

1. **BrainDB-Log:** Schreibt einen Backup-Eintrag mit Session-ID und Timestamp
2. **Relation-Extraktion:** Analysiert die bestehenden Fakten und extrahiert neue Entity-Verknuepfungen

### Regelbasierte Relation-Extraktion

Ein Python-Script analysiert die FactsDB und findet Verknuepfungen durch:

- **IP-Matching:** Wenn ein Value eine IP enthaelt, die einer anderen Entity gehoert
- **DNS-Referenzen:** Geraete die einen bestimmten DNS-Server nutzen
- **Entity-Referenzen:** Wenn ein Value den Namen einer bekannten Entity enthaelt
- **Namens-Prefixe:** "Laptop Alice" gehoert zur Entity "Alice"
- **Finanz-Verknuepfungen:** Versicherungs- und Eigentumsbeziehungen

UPSERT-Logik verhindert Duplikate bei wiederholter Ausfuehrung.

### Standardisierte Praedikate

Um den Knowledge Graph konsistent zu halten:

`nutzt`, `gehoert_zu`, `arbeitet_an`, `haengt_ab_von`, `ueberwacht_durch`, `erbt_von`, `verwaltet_durch`, `laeuft_auf`, `teil_von`, `konfiguriert_in`

## Ergebnis

- Wissensverlust durch Context-Komprimierung eliminiert
- Knowledge Graph waechst automatisch (17 → 28 Relationen durch erste Extraktion)
- Kein manueller Aufwand, keine vergessenen Sessions

## Kernprinzip

Das System wartet nicht darauf, gefragt zu werden. Es schuetzt sein Wissen aktiv, bevor es verloren gehen kann.
