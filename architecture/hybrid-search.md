# Hybrid Search

## Kernidee

Pure Vector Search versagt bei internen Fachbegriffen, Abkuerzungen und Projektnamen. Pure BM25 versteht keine Bedeutung. Die Loesung: Beide kombinieren, gewichten, und mit einem Zeitfaktor versehen.

## Architektur

### Gewichtung: 70% Vector + 30% BM25

- **Vector-Anteil (70%):** multilingual-e5-base Embeddings (768 Dimensionen). Versteht Bedeutung, multilingual, findet semantisch verwandte Dokumente. "Kostenanalyse" findet auch "Budget-Vergleich".
- **BM25-Anteil (30%):** Klassische Term-Frequency/Inverse-Document-Frequency Suche. Exakte Begriffe, Abkuerzungen, Projektnamen. "BV-KI" findet genau das.
- **Phrase-Bonus:** Zusammenhaengende Begriffe werden hoeher gewertet als einzelne Woerter. "Enterprise Knowledge Architecture" als Phrase rankt hoeher als Dokumente die diese Woerter einzeln enthalten.

### Warum 70/30?

Empirisch ermittelt. 80/20 gewichtete den BM25-Anteil zu schwach (Fachbegriffe gingen unter). 50/50 ueberbetonte exakte Matches auf Kosten semantischer Relevanz. 70/30 war der Sweet Spot fuer gemischte Queries (natuerlichsprachig + Fachbegriffe).

## Temporal Decay

### Problem

Ein Strategiepapier von 2023 rankte gleichwertig mit der aktuellen Version von 2026. Der Agent antwortete korrekt, aber mit veralteten Informationen.

### Loesung

Exponentieller Zerfall mit 90-Tage-Halbwertszeit:

```
final_score = relevance_score × e^(-λ × age_days)
λ = ln(2) / 90  (Halbwertszeit 90 Tage)
```

| Alter | Gewicht |
|---|---|
| 0 Tage (heute) | 100% |
| 30 Tage | 79% |
| 90 Tage | 50% |
| 180 Tage | 25% |
| 360 Tage | 6% |

### Evergreen-Ausnahmen

Bestimmte Dokumenttypen sind zeitlos und werden vom Decay ausgenommen:
- Grundsatzentscheidungen
- Architektur-Dokumentation
- Corporate Design Regeln
- Organisationsstrukturen
- Prozess-Definitionen

Markierung erfolgt per Metadaten im Dokument oder per Pfad-Regel (z.B. alles unter `/rules/` ist Evergreen).

## Embedding-Modell

**multilingual-e5-base** (768 Dimensionen)
- Multilingual: Deutsch + Englisch in derselben Suche
- Lokal ausfuehrbar: Keine API-Kosten, keine Daten verlassen den Rechner
- Qualitaet: Fuer die Dokumentenmenge (25k Chunks) ausreichend
- Einschraenkung: Modellwechsel erfordert kompletten Reindex

## Lessons Learned

1. **"Vector reicht" war falsch.** Drei Prototypen mit reiner Vector Search, alle gescheitert an Fachbegriffen.
2. **Phrase-Bonus war entscheidend.** Ohne ihn lieferte die Suche Wort-Suppe statt zusammenhaengender Ergebnisse.
3. **90 Tage Halbwertszeit empirisch.** Kuerzere Halbwertszeit (30 Tage) filterte zu viel relevantes Material. Laengere (180 Tage) liess zu viel Veraltetes durch.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
