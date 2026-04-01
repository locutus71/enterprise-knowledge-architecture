# Scope Isolation

## Kernidee

Verschiedene Kontexte brauchen verschiedene Daten. Arbeitsdaten duerfen nicht in private Projekte fliessen und umgekehrt. Aber ein persoenlicher Coach braucht das Gesamtbild. Loesung: Strikte Isolation mit selektivem Cross-Scope-Zugriff.

## Drei Scopes

### Scope 1: Arbeit

- Eigene Datenbank (Hot Memory, Deep Storage, Search Engine)
- Eigene Regeln und Konfigurationen
- Zugriff nur aus dem Arbeitskontext (bestimmt durch CWD)
- Keine Daten fliessen nach Privat

### Scope 2: Privat

- Komplett getrennte Infrastruktur
- Andere Pfade, andere Daten, andere Regeln
- Zugriff nur aus privatem Kontext
- Keine Daten fliessen in die Arbeit

### Scope 3: Coach

- Eigener Scope mit eigenem Storage
- **Einziger Cross-Scope-Zugriff:** Liest beide anderen Scopes
- Grund: Produktivitaet endet nicht an der Arbeit/Privat-Grenze
- Schreibt nur in seinen eigenen Scope

## Scope-Erkennung

Automatisch per Current Working Directory (CWD):
- `/Desktop/Work/` → Scope: Work
- `/Desktop/Personal/` → Scope: Personal
- `/Desktop/Coach/` → Scope: Coach

Keine manuelle Umschaltung. Der Kontext bestimmt den Scope.

## Isolation in der Praxis

Jeder Scope hat eigene Instanzen von:
- Hot Memory (andere Fakten)
- Deep Storage (andere Notes/Logs)
- Search Engine (andere Dokumente indexiert)
- Regeln (andere CLAUDE.md, andere Hooks)
- Fehler-Datenbank (andere BUGLIST)

## Coach als Cross-Scope-Bruecke

Warum der Coach beide Scopes lesen darf:
- Wenn die Arbeit 60h/Woche braucht, leidet das Private
- Wenn private Probleme Energie kosten, leidet die Arbeit
- Ein Coach der nur eine Seite sieht, kann nicht ganzheitlich beraten
- Lesezugriff reicht, Schreibzugriff ist nicht noetig

Der Coach schreibt seine Erkenntnisse in seinen eigenen Scope, nicht in Arbeit oder Privat.

## Designentscheidungen

1. **CWD-basiert statt manuell:** Weniger Fehlerquellen. Man kann nicht versehentlich im falschen Scope arbeiten.
2. **Coach liest, schreibt nicht:** Minimale Kopplung. Der Coach kann keine Arbeitsdaten veraendern.
3. **Keine gemeinsame Datenbank:** Lieber Duplikation als Kontamination. Wenn dasselbe Wissen in beiden Scopes relevant ist, existiert es zweimal.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
