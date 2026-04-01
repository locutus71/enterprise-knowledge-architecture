# 3-Layer Memory Architecture

## Kernidee

Wissen hat verschiedene Temperaturen. Aktuelle Fakten, destillierte Erkenntnisse und durchsuchbare Dokumente brauchen unterschiedliche Speicher mit unterschiedlichen Zugriffsmethoden.

## Die drei Schichten

### Layer 1: Hot Memory (<1ms)

Strukturierter Key-Value-Store fuer aktuelle Zustands-Fakten.

- **Zugriff:** Automatisch bei Session-Start in den Kontext injiziert
- **Inhalt:** Aktuelle Rollen, Status, IP-Adressen, Termine, Konfigurationen
- **Technologie:** SQLite mit einfachen Key-Value-Paaren
- **Aktualisierung:** Bei jeder relevanten Zustandsaenderung
- **Kapazitaet:** Hunderte Eintraege, gedeckelt auf ~2.000 Token bei Injektion

Kein Query noetig. Immer da. Der Agent weiss sofort wer fragt, was der aktuelle Stand ist, welche Regeln gelten.

### Layer 2: Deep Storage (~10ms)

SQLite-Datenbank mit 8 Tabellen fuer destilliertes Langzeitwissen.

- **Zugriff:** On-demand per Volltextsuche (FTS5) oder strukturierte Queries
- **Inhalt:** Notes (Erkenntnisse), Logs (Session-Protokolle), Snippets (Code), Configs, Relationen, Changelog
- **Technologie:** SQLite mit FTS5-Extension
- **Aktualisierung:** Am Ende jeder Session durch Destillation
- **Kapazitaet:** Tausende Eintraege, wachsend

Hier landet alles was aus Sessions destilliert wird: Debugging-Loesungen, strategische Entscheidungen, Fehler-Analysen, Recherche-Ergebnisse.

### Layer 3: Semantic Search (~100ms)

Hybrid Search Engine ueber alle Projektdokumente.

- **Zugriff:** On-demand per natuerlichsprachiger Suche
- **Inhalt:** Alle Projektdokumente als Chunks (500-1000 Token)
- **Technologie:** sqlite-vec (Vector) + BM25 (Keyword), multilingual-e5-base Embeddings (768 Dimensionen)
- **Aktualisierung:** Periodischer Reindex (getriggert durch Session-Stop Hook)
- **Kapazitaet:** 25.000+ Chunks aus 400+ Dateien

Findet Zusammenhaenge die der Agent nicht explizit kennt. "Was haben wir letztes Quartal zu X entschieden?" funktioniert, auch wenn der Agent bei der Entscheidung nicht dabei war.

## Warum drei Schichten?

Die CPU-Cache-Analogie: L1 ist schnell und klein (Register/aktuelle Fakten), L2 ist mittelschnell und strukturiert (Cache/destilliertes Wissen), L3 ist langsamer aber umfassend (RAM/alle Dokumente).

Der Fehler den viele machen: Alles in einen Vector Store. Das ist wie RAM ohne Cache. Funktioniert, aber jeder Zugriff auf "Wie heisst der aktuelle Ansprechpartner?" kostet 100ms statt <1ms und verbraucht Retrieval-Budget fuer triviale Fakten.

## Datenfluss zwischen den Schichten

```
Session-Arbeit
    ↓ Destillation (Session-Ende)
Deep Storage (Layer 2)
    ↓ Export (export_for_qualia.py)
Markdown-Dateien
    ↓ Reindex (automatisch via Stop-Hook)
Semantic Search (Layer 3)
    ↓ Naechste Session: bessere Suchergebnisse
```

Hot Memory (Layer 1) wird direkt aktualisiert wenn sich Fakten aendern, unabhaengig vom Destillations-Zyklus.

## Designentscheidungen

1. **SQLite statt PostgreSQL/Redis:** Ein System, keine Infrastruktur. Laeuft auf Laptop und Server identisch.
2. **FTS5 statt Elasticsearch:** Reicht fuer die Datenmenge. Keine zusaetzliche Infrastruktur.
3. **Lokale Embeddings statt API:** multilingual-e5-base lokal, keine Daten verlassen den Rechner.
4. **Token-Budget bei Injektion:** Hot Memory auf ~2.000 Token begrenzt. Mehr Kontext ist nicht immer besser.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
