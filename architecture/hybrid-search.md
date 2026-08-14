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

### Score-Normalisierung: der Schritt, ohne den die Gewichtung nichts bedeutet

Die beiden Arme liefern nicht vergleichbare Zahlen. Cosine-Aehnlichkeit liegt
zwischen 0 und 1, BM25 ist nach oben offen und haengt von Korpus- und
Query-Statistik ab. Wer beide ungewichtet addiert, bekommt kein 70/30, sondern
ein Mischungsverhaeltnis, das je Query schwankt. Die Gewichtung waere dann kein
Designparameter, sondern Zufall.

Vor der Fusion wird der BM25-Arm deshalb auf denselben Wertebereich gebracht.
Zwei Wege, beide im Einsatz:

- **Min-Max ueber die Trefferliste** der aktuellen Query, oder
- **Division durch einen festen Normierungsfaktor** mit Deckelung bei 1,
  `min(bm25 / bm25_norm_factor, 1.0)`.

Erst danach gilt `score = vec_weight * vec + bm25_weight * bm25`.

### Stand der Implementierung (August 2026)

Die Abschnitte oben beschreiben die Herleitung von Maerz 2026. Der laufende
Stand weicht davon ab:

| Parameter | Dokumentiert oben | Laufender Stand |
|---|---|---|
| Vector / BM25 | 70 / 30 | **90 / 10** |
| BM25 k1 / b | nicht genannt | 2.62 / 0.62 |
| Normierungsfaktor | nicht genannt | 9.74 |
| Phrase-Bonus | qualitativ | Faktor 1.64 |

Der BM25-Anteil wird nicht getrennt konfiguriert, sondern als
`1.0 - vec_weight` abgeleitet. Ein zusaetzlich gesetztes Gewichtsfeld bleibt
deshalb wirkungslos.

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

### Vom Schalter zur Skala (Stand August 2026)

Die binaere Aufteilung oben, Decay an oder aus, hat sich als zu grob erwiesen.
Der laufende Stand kennt vier Zerfallsklassen, jede mit eigener Halbwertszeit
und einem Boden, unter den der Score nicht faellt:

| Klasse | Halbwertszeit | Boden |
|---|---|---|
| ephemeral | 14 Tage | 0.05 |
| standard | 90 Tage | 0.10 |
| strategic | 365 Tage | 0.30 |
| permanent | kein Zerfall | 1.00 |

Die Zuordnung geschieht automatisch: ueber den Dokumenttyp (Logs werden
ephemeral, Konfigurationen permanent) und ueber Schlagworte (Strategie,
Vertrag, Architektur, Frist, Entscheidung heben nach strategic; Identitaet,
Relation, Adresse heben nach permanent). Der Boden ist der eigentliche
Unterschied zur Evergreen-Ausnahme: ein altes Dokument verschwindet nicht,
es rutscht nur nach unten.

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
