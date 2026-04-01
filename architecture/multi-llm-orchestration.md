# Multi-LLM Orchestrierung

## Kernidee

Nicht ein Modell fuer alles, sondern das richtige Modell fuer die richtige Aufgabe. Cloud fuer Komplexes, lokal fuer Schnelles. Kein Vendor Lock-in.

## Zwei Ebenen

### Lokal (On-Premise)

Kleine, schnelle Modelle fuer einfache Aufgaben:
- **Modell:** qwen2.5:3b via Ollama
- **Aufgaben:** Klassifikation, Routing, Embedding-Vorbereitung, einfache Textverarbeitung
- **Vorteile:** 0 EUR Kosten, keine Latenz, volle Datenkontrolle, DSGVO-konform
- **Hardware:** Laeuft auf normalem Laptop oder VPS (2 vCPU, 8GB RAM reichen)

### Cloud (API)

Grosse Modelle fuer komplexe Aufgaben:
- **Claude Opus:** Komplexe Implementierung, Architektur-Entscheidungen, lange Analysen
- **Claude Sonnet:** Recherche, Sub-Agents, schnelle Aufgaben
- **GPT-4:** Alternative Perspektive bei kritischen Entscheidungen
- **Gemini:** Grosse Kontextfenster (1M+ Token)
- **Perplexity:** Web-Recherche mit Quellenangabe
- **Grok:** Echtzeit-Informationen

## Auto-Router

Entscheidet automatisch: Lokal oder Cloud? Welches Cloud-Modell?

Entscheidungskriterien:
1. **Komplexitaet:** Einfache Klassifikation → lokal. Strategische Analyse → Cloud/Opus.
2. **Datensensitivitaet:** Personenbezogene Daten → lokal. Oeffentliche Recherche → Cloud.
3. **Kosten:** Bulk-Operationen (100 Dokumente klassifizieren) → lokal. Einzelne tiefe Analyse → Cloud.
4. **Geschwindigkeit:** Echtzeit-Routing → lokal. Kann warten → Cloud.

## Cross-LLM Benchmarking

Kritische Entscheidungen werden ueber mehrere Modelle validiert.

Nicht "was sagt Claude?" sondern "stimmen Claude, GPT und Gemini ueberein?".

Ablauf:
1. Dieselbe Frage an 2-3 verschiedene Modelle
2. Ergebnisse vergleichen
3. Uebereinstimmung = hohe Confidence
4. Divergenz = Signal fuer tiefere Pruefung

Divergenzen sind kein Problem, sondern ein Feature. Wenn Modelle sich widersprechen, hat mindestens eins etwas entdeckt das die anderen uebersehen haben.

## Designentscheidungen

1. **Kein Vendor Lock-in:** Jedes Modell ist austauschbar. Die Architektur haengt nicht von einem Anbieter ab.
2. **Lokal-First fuer Sensitive Daten:** Personenbezogene Daten verlassen den Rechner nicht.
3. **Sub-Agents immer mit Sonnet:** Schneller, guenstiger, ausreichend fuer Teilaufgaben.
4. **Opus nur fuer die Hauptaufgabe:** Kosten-Effizienz durch gezielte Modellwahl.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
