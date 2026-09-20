# Harness Evolution: Validierung fuer Aenderungen am Harness

## Kernidee

Die Validation Layer prueft Wissen, das durch den Harness fliesst. Sie prueft nicht
den Harness selbst: CLAUDE.md, Hook-Logik, Hot-Memory-Inhalte, BUGLIST, Suchparameter,
Decay-Klassen. Diese Schicht aendert sich laufend, aber kein Gate prueft die Aenderung
gegen eine Baseline, kein Snapshot erlaubt Rollback.

Harness Evolution schliesst diese Luecke: Aenderungen am Harness durchlaufen dieselbe
Zero-Trust-Logik wie Aenderungen am Wissen. Vorlage ist HarnessEvolve
(arXiv:2609.00829, September 2026), uebertragen auf ein Ein-Personen-System mit
SQLite und lokalen Modellen.

## Das Problem: Der Harness evolviert ungeprueft

Zwei Stellen in dieser Dokumentation zeigen, dass das Problem real ist:

- **Parameter-Drift.** Die Hybrid-Search-Gewichtung ist von 70/30 auf 90/10 gewandert,
  k1, b und Normierungsfaktor kamen dazu (`hybrid-search.md`, Stand August 2026). Die
  Aenderung ist dokumentiert als Abweichung, nicht als gemessene Verbesserung.
- **Offene Rueckkopplung.** Dual Decay zaehlt Sichtbarkeit statt Nutzen: `last_accessed`
  wird bei jedem Suchtreffer gesetzt, nicht erst, wenn der Chunk in eine Antwort eingeht
  (`dual-decay.md`, Abschnitt "Bekannte Grenze"). Das System weiss nicht, welcher Treffer
  die Antwort getragen hat.

Beides sind Faelle, in denen der Harness veraendert wurde oder sich selbst veraendert,
ohne dass ein Gate die Wirkung geprueft haette.

## Drei Fehlerarten selbst-evolvierender Systeme

HarnessEvolve benennt drei Fehlerarten. Alle drei treten hier auf:

| Fehlerart | Bedeutung | Wo sie hier auftritt |
|---|---|---|
| Credit Assignment Failure | Feedback ist terminal (Antwort richtig / falsch); unklar, welcher Schritt den Fehler verursacht hat | Der Stop-Hook destilliert "Was wurde gelernt?", kann aber nicht sagen, ob ein Retrieval-Miss, ein falscher Fakt oder ein Reasoning-Fehler vorlag |
| Shortcut Learning | Das System memoriert task-spezifische Muster statt generalisierbarer Regeln | Eine Regel in CLAUDE.md, die nur den letzten Vorfall abfaengt; BUGLIST-Eintraege, die als Liste einzelner Symptome in den Kontext injiziert werden |
| Catastrophic Forgetting | Ein Update verschlechtert bereits erworbene Kompetenz | Parameter-Drift ohne Regressionspruefung; Hot-Memory-Aenderungen ohne Blick auf das Token-Budget |

## Sechs Bausteine

### 1. Trajectory-Logging: Nutzung statt Sichtbarkeit

Eine Tabelle `trajectories` in Deep Storage, geschrieben vom Stop-Hook und vom
PreCompact-Hook:

```
session_id | step | query | chunks_returned[] | chunks_cited[] | tool | outcome
```

`chunks_cited` ist die Menge der Chunks, die der Agent in der Antwort tatsaechlich
referenziert hat. Source-Pinning (Gate 1) liefert das bereits: `source=file://pfad`
in der Antwort ist der Zitat-Nachweis.

Damit wird die offene Rueckkopplung in Dual Decay geschlossen:

```
decay_usage = exp(-lambda_2 * tage_seit_letzter_ZITIERUNG)   # nicht: seit letztem Treffer
```

Trajectory-Logging ist Voraussetzung fuer alle weiteren Bausteine.

### 2. Referenz-Trajektorien aus gelabelten Faellen

HarnessEvolve lokalisiert Fehler, indem es eine Referenz-Trajektorie mit bekannter
richtiger Antwort erzeugt und die fehlgeschlagene Ausfuehrung dagegen ausrichtet. Das
setzt Labels voraus: Faelle, in denen die richtige Antwort bekannt ist.

In den meisten Sessions gibt es kein Label. An drei Stellen entsteht aber nachtraeglich
eines, immer vom User, nie vom Agenten:

| Stelle | Was passiert | Warum das ein Label ist |
|---|---|---|
| Gate 4, Konfliktaufloesung | Agent-Fakt widerspricht bestehendem Fakt, User entscheidet | Die Entscheidung ist die Ground Truth; die verworfene Version ist ein gelabelter Fehler |
| `confidence=verified` | Primaerquelle, vom User bestaetigt | Label fuer alle spaeteren Antworten, die diesen Fakt haetten nutzen muessen |
| BUGLIST-Eintrag geloest | Ein Fehler wird behoben | Die Behebung ist die richtige Antwort auf die urspruenglich falsch beantwortete Frage |

Bei jedem dieser Ereignisse entsteht ein Datensatz
`(urspruengliche Frage, fehlgeschlagene Trajektorie, richtige Antwort)`. Ein Offline-Job
fuehrt die Frage erneut aus, diesmal mit der Antwort im Kontext, und vergleicht beide
Trajektorien. Der Diff ordnet den Fehler einer von vier Klassen zu:

| Fehlerklasse | Diagnose aus dem Trajektorien-Diff | Zustaendige Komponente |
|---|---|---|
| Retrieval-Miss | Referenz zitiert Chunk X, Original hatte X nicht in Top-k | Hybrid Search, Decay-Klasse |
| Retrieval-Ignored | X war in Top-k, wurde nicht zitiert | Prompt, Hot-Memory-Anweisung |
| Store-Fehler | X wurde zitiert, X ist falsch | Validation Gates 3 bis 5 |
| Reasoning-Fehler | Gleiche Chunks, andere Schlussfolgerung | Modell-Routing (Multi-LLM) |

Der Vergleich ist deterministisch (Chunk-IDs), kein LLM noetig. Die Referenz-Trajektorie
dient nur als Diff-Basis, nicht als Vorbild; dass ein Agent mit vorgegebener Antwort das
Reasoning abkuerzt, ist deshalb unkritisch.

Nur die gelabelte Teilmenge speist die Evolution. Die uebrigen Sessions werden weiter
destilliert wie bisher, aber sie loesen keine Harness-Aenderung aus. Das verhindert, dass
der Agent aus ungeprueften Sessions Regeln fuer sich selbst schreibt.

### 3. Fehler-Clustering statt BUGLIST-Liste

Die BUGLIST wird bei SessionStart in das 2.000-Token-Budget injiziert. Eine wachsende
Liste einzelner Vorfaelle ist Prompt Bloat. Die Fehlersignale aus Baustein 2 werden
geclustert (Schluessel: Fehlerklasse plus betroffener Pfad oder Begriff; wenn noetig
ueber das lokale 3B-Modell). Injiziert werden Muster mit Haeufigkeit, nicht Vorfaelle.
Die Cluster sind zugleich die Eingabe fuer Harness-Aenderungen.

### 4. Gate 8: Harness-Validierung

Gates 1 bis 7 pruefen Fakten. Gate 8 prueft Aenderungen an CLAUDE.md, Hook-Skripten,
Suchparametern, Decay-Klassen und Hot-Memory-Regeln. Es hat zwei Haelften.

**Quality-Haelfte (deterministisch, kein LLM):**

- Token-Delta der Aenderung gegen das Budget. Hot Memory ist auf ~2.000 Token gedeckelt;
  das ist die Grenze.
- Leakage-Check: Enthaelt die neue Regel Begriffe, die nur in der Ground Truth des
  ausloesenden Falls vorkommen? Dann ist es eine memorierte Antwort, keine Regel.
  Das ist Gate 3 (Keyword Cross-Check), invertiert.

**Performance-Haelfte:**

Der Synthetic Test Harness (MRR, Recall@5, p95/p99, kontrollierte Ground Truth) liefert
die Messung. Bisher dient er Parameter-Sweeps. Als Gate laeuft er vor jeder
Harness-Aenderung: Verbesserung auf dem ausloesenden Fehler-Cluster, keine Regression auf
den letzten N Clustern.

Die Parameter-Drift aus `hybrid-search.md` waere damit kein dokumentierter Abstand,
sondern ein akzeptierter Snapshot mit Messwert.

### 5. Snapshot-Pool mit `superseded_by`

`dual-decay.md` schlaegt `superseded_by`-Ketten fuer Dokumente vor. Dasselbe Konzept
fuer den Harness: CLAUDE.md, Hook-Konfiguration und Suchparameter (`vec_weight`, k1, b,
Normierungsfaktor, Decay-Klassen) werden als versionierte Einheit in Deep Storage
abgelegt. Jede Version traegt ihren Gate-8-Messwert und `superseded_by`. Der
SessionStart-Hook laedt den zuletzt akzeptierten Snapshot, nicht die zuletzt editierte
Datei. Rollback ist ein Pointer-Wechsel.

### 6. Entkoppelter Evaluator

Gate 7 fordert fuer die Cross-Model-Synthese eine frische Sitzung ohne Kontext aus den
Einzel-Sitzungen (`validation-layer.md`, "Warum frische Sitzung"). HarnessEvolve trennt
Execution und Evaluation aus demselben Grund; die Ablation des Papers zeigt, dass diese
Trennung der wirksamste Einzelfaktor ist.

Uebertragen: Der Trajektorien-Diff (Baustein 2) und die Regel-Vorschlaege aus den Clustern
(Baustein 3) laufen in einer frischen Session ohne Kontext der Ursprungssession. Der
Agent, der den Fehler gemacht hat, schreibt nicht seine eigene Regel dagegen.

## Rollen fuer den Verbesserungszyklus

Planer/Worker deckt zwei der vier Rollen ab, die HarnessEvolve trennt. Die beiden anderen
sind die, die der Validation Layer fordert: Wer ausfuehrt, bewertet nicht; wer plant,
laesst nicht selbst zu.

```
Fehler-Cluster --> Planer --> Pakete --> Worker (je eigener Worktree, eigene Session)
                                              |
                                              v
                                         Evaluator (frische Session, sieht den Plan nicht)
                                              |
                                              v
                                          Gate 8 --> Snapshot-Pool --> Freigabe durch User
```

| Rolle | Aufgabe | Modell (nach `multi-llm-orchestration.md`) |
|---|---|---|
| Planer | Liest Cluster, formuliert Pakete mit messbarem Ziel ("Recall@5 auf Cluster X von 0,61 auf >= 0,75, keine Regression auf Y, Z") | Opus oder gleichwertig |
| Worker | Aendert einen Harness-Bestandteil pro Paket | Sonnet ("Sub-Agents immer mit Sonnet") |
| Evaluator | Test Harness gegen Worker-Output, Trajektorien-Diff | deterministisch wo moeglich, sonst Sonnet in frischer Session |
| Gating | Gate 8 | kein LLM |

Zwei Regeln:

1. **Ein Paket, eine Aenderung, ein messbares Kriterium.** Ein Kriterium, das nur der
   Planer selbst beurteilen kann, ist keins.
2. **Der Evaluator sieht den Plan nicht.** Er bekommt Worker-Output und Testdaten, nicht
   die Begruendung. Sonst bewertet er die Ueberzeugungskraft des Plans statt das Ergebnis.

Der Zyklus laeuft in einem eigenen Scope (`scope-isolation.md`): Lesezugriff auf den
Ziel-Scope, Schreibzugriff nur auf den Snapshot-Pool. Wie der Coach: liest, schreibt nur
in den eigenen Scope. Ein akzeptierter Snapshot wird bewusst uebernommen, nicht
automatisch.

## Priorisierung

| Reihenfolge | Baustein | Baut auf | Aufwand |
|---|---|---|---|
| 1 | Trajectory-Logging | Stop/PreCompact-Hook, Gate 1 | klein |
| 2 | Gate 8 | Synthetic Test Harness, Gate 3 | mittel |
| 3 | Referenz-Trajektorien | Baustein 1, Gate 4, `confidence=verified` | mittel |
| 4 | Snapshot-Pool | Gate 8, `superseded_by` | klein |
| 5 | Fehler-Clustering | Baustein 3, lokales 3B-Modell | klein |
| 6 | Entkoppelter Evaluator | Baustein 3, Gate-7-Prinzip | trivial (Konfiguration) |

## Grenzen

- **Kosten.** Vier Rollen pro Update waeren fuer ein Ein-Personen-System zu teuer.
  Deshalb sind Trajektorien-Diff und Quality-Haelfte deterministisch; nur Clustering und
  Regel-Vorschlag brauchen ein Modell, und das kann lokal laufen.
- **Portabilitaet.** Die Evolutions-Pipeline darf nicht die sechste hook-gebundene
  Komponente werden. Sie laeuft als Offline-Batch auf Deep Storage (Python, SQLite),
  vom Stop-Hook getriggert, aber nicht davon abhaengig.
- **Labels sind selten.** Die Pipeline arbeitet mit vielleicht fuenf Prozent der
  Sessions. Das ist gewollt: Nur nachweislich fehlgeschlagene Faelle mit bekannter
  Korrektur speisen die Evolution.

## Status

Konzept, September 2026. Nichts davon ist implementiert. Trajectory-Logging ist der
erste Schritt, weil es die offene Rueckkopplung in Dual Decay schliesst und alles Weitere
voraussetzt.

Quelle: HarnessEvolve: Learning from Reference Trajectories for Reliable Agent
Self-Evolution, arXiv:2609.00829 (September 2026).

---
Erstellt: 20.09.2026 | Status: Konzept
