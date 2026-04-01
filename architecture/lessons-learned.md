# Lessons Learned

## Was nicht funktioniert hat

### "Ein Vector Store reicht"

Drei Prototypen gebaut, alle mit reiner Vector Search. Interne Fachbegriffe, Abkuerzungen, Projektnamen gingen verloren. "BV-KI" lieferte keine Ergebnisse. Die Loesung war nicht ein besseres Embedding-Modell, sondern Hybrid Search: 70% Vector + 30% BM25 mit Phrase-Bonus.

### "LLMs validieren sich selbst"

Ein LLM das seine eigenen Aussagen prueft, bestaetigt sich in der Regel selbst. Validation braucht externe Checks: Dateisystem pruefen, Fachbegriffe abgleichen, Widersprueche zu bestehendem Wissen erkennen. Erst mit den 6 Validation Gates sank die Fehlerquote spuerbar.

### "Cloud-First ist immer besser"

Fuer Klassifikation und Routing ist ein lokales 3B-Modell schneller, guenstiger und datenschutzkonformer als ein Cloud-API-Call. Cloud fuer Komplexes, lokal fuer Schnelles. Nicht entweder-oder.

### "Mehr Kontext macht bessere Antworten"

200k Token Kontextfenster vollstopfen macht Antworten nicht besser, sondern langsamer und teurer. Die Loesung: Schichtung. Nur das Noetigste automatisch laden (Hot Memory ~2.000 Token), den Rest on-demand suchen.

### "Theoretische Frameworks beeindrucken"

Content-Validierung via LinkedIn: Ein theoretischer Post ueber eine Kontext-Layer-Architektur (VKLA) bekam 64 Impressions in einer Woche. Ein Praxis-Post ueber dieselben Konzepte, formuliert als Erfahrungsbericht mit konkreten Fehlern und Loesungen, bekam 2.215+ in 26 Stunden. Verhaeltnis 35:1. Dasselbe Wissen, andere Verpackung.

## Was ueberraschend gut funktioniert hat

### Policy as Code via Hooks

Regeln in Dokumentation werden vergessen. Regeln als automatisierte Hooks werden erzwungen. Email-Guard, Delete-Block, Kontext-Laden, Wissen-Sichern. Das System braucht keine Disziplin.

### Temporal Decay mit Evergreen-Ausnahmen

Simples Konzept, grosse Wirkung. 90-Tage-Halbwertszeit loest 80% des "veraltete Informationen"-Problems. Evergreen-Markierung fuer den Rest.

### Knowledge Loop als Architektur-Konsequenz

Nicht als Feature geplant, sondern als Nebeneffekt der Hook-Architektur entstanden. SessionStart laedt Kontext, Stop sichert Wissen, Reindex macht es durchsuchbar. Der Loop ergab sich von selbst.

### Cross-LLM Benchmarking

Wenn Claude, GPT und Gemini sich einig sind, ist die Confidence hoch. Wenn nicht, hat mindestens eins etwas entdeckt. Divergenzen sind wertvoller als Uebereinstimmungen.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
