# Single Source of Truth

## Kernidee

Dieselbe Information darf nicht an zwei Orten gepflegt werden. Eine falsche
Information korrigiert man einmal. Zwei Staende laufen dagegen mit jeder
Aenderung weiter auseinander, und der Agent erwischt mal den einen, mal den
anderen. Das bleibt meist unbemerkt, weil beide Staende plausibel aussehen.

Die uebrige Architektur erkennt Duplikate gut. Sie beantwortet aber nicht die
Frage, welcher Ort gilt. Dieses Dokument beschreibt die fehlende Regel: Jeder
Informationstyp hat genau ein Zuhause, und ueberall sonst steht hoechstens ein
Verweis.

## Das Problem

Typischer Verlauf: Ein Team beschliesst, seine Doku kuenftig im Ordnersystem
zu pflegen. Das Wiki bleibt per Konnektor angebunden, dort liegt die alte Doku
weiter. Ein Teil des Teams pflegt das Wiki, der andere den Ordner. Nach ein
paar Wochen gibt es zwei Staende.

Der Agent verstaerkt das, weil er lokal schnell und guenstig sucht, ueber einen
Konnektor dagegen langsam und teuer. Die meisten falschen Kopien entstehen
ohnehin durch den Agenten selbst, wenn er nicht weiss, wohin eine Information
gehoert:

- Nach dem Lesen eines Meeting-Transkripts speichert er eine Notiz mit der
  halben Zusammenfassung.
- Er schreibt den Stand aus dem Ticketsystem in eine Markdown-Datei, um ihn
  beim naechsten Mal schneller zu finden.

## Warum Metadaten allein nicht reichen

Source-Pinning (Gate 1), Autor und Zeitstempel sagen, **woher** eine
Information kommt und **wann** sie entstand. Sie sagen nicht, **wo sie gilt**.

| Mechanismus | Was er leistet | Wo er beim Duplikat-Problem versagt |
|---|---|---|
| Gate 1: Source-Pinning | Herkunft jedes Fakts bekannt | Zwei gepflegte Orte haben beide eine gueltige Quelle |
| Autor / Zeitstempel | Wer, wann | Beide Staende sind frisch und haben echte Autoren |
| Gate 4: Contradiction Check | Widerspruch flaggen, User entscheidet | Prueft nur gegen lokales Wissen; Systeme hinter Konnektoren liegen nicht im Index |
| Gate 6: Export-Filter | Duplikate vor dem Index entfernen | Entfernt Dubletten im eigenen Export, nicht Kopien externer Systeme |
| Temporal Decay | Neueres rankt hoeher | Die Kopie von gestern schlaegt das Original von letzter Woche |
| Dual Decay | Genutztes bleibt sichtbar | Die lokale Kopie wird oft gefunden und dadurch immer wieder verjuengt |

Hinzu kommt, dass die Architektur selbst Kopien erzeugt: Destillation am
Session-Ende, Export aus Deep Storage nach Markdown, Reindex. Und Scope
Isolation nimmt Duplikation zwischen Scopes bewusst in Kauf ("Lieber
Duplikation als Kontamination"). Dort ist das gewollt, innerhalb eines Scopes
ist es ein Fehler.

## Loesung

### 1. Zuhause-Register pro Informationstyp

Fuer jeden Informationstyp wird festgelegt, wo er lebt. Das Register steht in
der CLAUDE.md des Scopes, damit der Agent es bei jeder Session kennt:

| Informationstyp | Zuhause | Lokal erlaubt |
|---|---|---|
| Meeting-Inhalte (Transkript) | Transkript-Tool | Verweis |
| Meeting-Ergebnisse (Entscheidungen, Aufgaben) | Deep Storage | Inhalt |
| Ticket-Status | Ticketsystem | Verweis |
| Projekt-Doku | Ordnersystem *oder* Wiki, nie beides | je nach Festlegung |
| Aktuelle Zustands-Fakten | Hot Memory | Inhalt |

Entscheidend ist die Unterscheidung beim Meeting: Das Transkript ist eine
Kopie dessen, was im Tool liegt. Was daraus folgt (Entscheidungen, Aufgaben),
ist eine neue Information und darf lokal gespeichert werden.

Das Register regelt drei Dinge: wo eine Information liegt, wohin neue
Informationen dieses Typs geschrieben werden, und was lokal **nicht** abgelegt
wird.

### 2. Metadatum `system_of_record`

Ergaenzend zu `source` bekommt jeder Eintrag das Feld `system_of_record`:

```
source=file://notes/sprint-42.md
system_of_record=ticket://PROJ-42
```

`source` sagt, woher der Agent die Information hat. `system_of_record` sagt,
wo sie gilt. Stimmen beide nicht ueberein, ist der Eintrag eine Kopie.

### 3. Home-Check beim Schreiben

Der Home-Check laeuft beim Persistieren, zusammen mit den Validation Gates. Er
ist deterministisch, kein LLM noetig:

```
Agent will Eintrag schreiben
    |
    ├── Typ im Register nachschlagen
    |
    ├── Zuhause = dieser Speicher?  ──→ Inhalt speichern
    |
    └── Zuhause = externes System?  ──→ nur Verweis speichern
                                        (system_of_record + Abrufzeitpunkt)
```

Ein Verweis enthaelt keinen Inhalt, nur Ziel, Abrufzeitpunkt und eine
Kurzbeschreibung fuer die Suche. Damit findet die Hybrid Search den Verweis und
fuehrt den Agenten zum Original, statt ihm einen veralteten Stand zu liefern.

### 4. Unvermeidbare Kopien als Snapshot

Manche Kopien lassen sich nicht vermeiden, etwa wenn ein externes System nur
langsam erreichbar ist. Sie werden als Snapshot markiert und der Zerfallsklasse
`ephemeral` zugeordnet (14 Tage Halbwertszeit, Boden 0.05, siehe
[Hybrid Search](hybrid-search.md)). Sie rutschen schnell nach unten, statt neben
dem Original weiterzuleben.

### 5. Versionskette statt Statistik

Wird ein Dokument ersetzt, verweist die alte Fassung per `superseded_by` auf die
neue. Die Frage "welche Fassung gilt?" ist eine Versionsfrage, keine
Alterungsfrage (siehe [Dual Decay](dual-decay.md)). Ersetzte Fassungen werden
wie `confidence=rejected` vom Export ausgeschlossen.

### 6. Regelmaessiger Duplikat-Audit

Als Teil des regelmaessigen Aufraeumens:

1. Chunk-Paare mit sehr hoher Embedding-Aehnlichkeit, aber verschiedener
   `source` oder verschiedenem `system_of_record` sammeln.
2. Lokale Eintraege suchen, deren Typ laut Register ein externes Zuhause hat.
3. Fuer jeden Treffer entscheidet der User, wo das Zuhause ist. An der anderen
   Stelle bleibt hoechstens ein Verweis.

Das Ergebnis fliesst zurueck ins Register. So wird aus einer Einzelentscheidung
eine Regel.

## Zusammenspiel mit anderen Komponenten

- **Validation Layer** prueft, ob eine Information stimmt. Der Home-Check prueft,
  ob sie an diesen Ort gehoert.
- **Hybrid Search** findet Verweise wie normale Eintraege und fuehrt zum Original.
- **Temporal / Dual Decay** behandeln Snapshots ueber die Klasse `ephemeral`.
- **Scope Isolation** bleibt unberuehrt: Duplikation *zwischen* Scopes ist
  gewollt, die Regel gilt *innerhalb* eines Scopes.

## Status

Konzept, Oktober 2026. Nicht implementiert. Bereits vorhanden sind die
Bausteine: Source-Pinning, Export-Filter, Zerfallsklassen. Offen sind das Feld
`system_of_record`, der Home-Check, die `superseded_by`-Kette und der
Duplikat-Audit.

## Offene Designfragen

- Wie wird ein Verweis aktuell gehalten, wenn das Ziel umzieht oder geloescht
  wird? (Kandidat: Existence Check, Gate 2, auch auf Verweise anwenden)
- Ab welcher Embedding-Aehnlichkeit gilt ein Paar als Duplikat und nicht als
  verwandter Inhalt?
- Soll der Home-Check Kopien hart ablehnen oder automatisch in Verweise
  umwandeln?

---
Erstellt: 27.10.2026 | Autor: Holger Woelfle
