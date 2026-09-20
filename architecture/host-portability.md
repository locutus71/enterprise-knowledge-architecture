# Host Portability: Modell und Host tauschen, Wissensschicht behalten

## Kernidee

Das Modell ist heute schon austauschbar. Der Host ist es nicht: Fuenf Komponenten
(Hook System, Timestamp Hook, Incremental Extraction, der automatische Teil von Knowledge
Loop und Scope Isolation) haengen an der Hook-API von Claude Code (siehe README,
"Scope of portability").

Host Portability zerlegt diese Kopplung in zwei Haelften, die verschieden portabel sind,
und macht die Wissensschicht zu dem Teil, der bleibt, wenn Modell oder Host wechseln.

## Zwei Lesarten von "Harness behalten"

| Lesart | Was "Harness" meint | Was getauscht wird |
|---|---|---|
| A | Claude Code samt der eigenen Hooks | nur das Modell dahinter |
| B | die eigene Schicht (Hooks, Memory, Gates, Loop) | das Host-Produkt und das Modell |

Beide sind loesbar. A ist der schnelle Weg, B der robuste.

## Loesung A: Modell tauschen, Host behalten

Claude Code spricht mit einem Anthropic-kompatiblen API-Endpunkt. Ein LLM-Gateway
(z.B. LiteLLM) dazwischen uebersetzt in andere Provider-Formate:

```
Claude Code --ANTHROPIC_BASE_URL--> Gateway --> beliebiges Modell mit Tool-Calling
    |
    +-- alle fuenf Hooks: unveraendert
```

Ergebnis: 100 Prozent der Harness-Funktionalitaet, kein Umbau. Grenzen:

- Claude Code bleibt die Abhaengigkeit.
- System-Prompt und Tool-Schema von Claude Code sind auf Claude abgestimmt. Ein fremdes
  Modell bekommt einen Prompt, der fuer ein anderes Modell geschrieben wurde. Bei
  Modellen mit eigenem, gut abgestimmtem Host (etwa Astra in Codex CLI) ist das eine
  bewusste Verschlechterung.
- Ein Nicht-Claude-Modell hinter dem Gateway ist ein Community-Pattern, keine
  dokumentierte Konfiguration. Extended Thinking, Prompt Caching und Beta-Header mappen
  nicht 1:1.

Loesung A ist richtig fuer Modelle ohne eigenen guten Host (DeepSeek, Qwen, lokal).

## Loesung B: Die eigene Schicht vom Host loesen

Die fuenf gekoppelten Komponenten nutzen den Host fuer zwei verschiedene Dinge:

1. **Push.** Der Host erzwingt etwas: Kontext injizieren, Tool blockieren, vor
   Kompression sichern. Das ist Policy as Code (`hook-system.md`).
2. **Pull.** Der Agent holt etwas: Suche, Deep-Storage-Abfrage, Fakten schreiben.

Pull ist heute fast portabel, wird nur nicht so verpackt. Push ist der harte Teil.

### Pull: MCP-Server

Hybrid Search, Deep Storage, Hot Memory und die Gates als MCP-Server. MCP wird von Claude
Code, Codex CLI, OpenCode, Gemini CLI und den IDE-Agenten gesprochen. Ein Server, alle
Hosts, jedes Modell:

```
search(query, scope)      --> Layer 3
recall(topic)             --> Layer 2
remember(fact, source)    --> Gates 1 bis 6, dann Layer 1 oder 2
conflicts()               --> offene Gate-4-Konflikte
now()                     --> Timestamp
```

Reines Python und SQLite hinter einem Standardprotokoll. Groesster Hebel, geringstes
Risiko, sofort auch in Claude Code nutzbar.

### Push: Adapter-Contract

Die fuenf Events als eigener Vertrag: JSON auf stdin, exakt wie Claude Code es heute
liefert, aber als Spezifikation, die dem System gehoert und nicht dem Host.

```
session_start(cwd, session_id)   --> Kontext-Text
prompt_submit(text)              --> Injektions-Text
pre_tool(tool, args)             --> allow | block
pre_compact(transcript)          --> void
stop(session_id, transcript)     --> void
```

Pro Host ein duenner Adapter, der Host-Events auf diesen Vertrag abbildet. Claude Code:
1:1. Andere Hosts: Plugin-API, wo vorhanden. Was ein Host nicht liefert, faellt auf einen
Fallback zurueck:

| Event | Portabler Mechanismus | Verlust gegenueber Claude Code |
|---|---|---|
| `session_start` | Wrapper-Skript startet den Host und generiert vorher einen Abschnitt in `AGENTS.md` (de-facto-Standard, den Codex CLI, OpenCode, Gemini CLI lesen) | keiner |
| `stop` | Wrapper-Skript nach Prozessende | keiner |
| Scope per CWD | Wrapper liest CWD, uebergibt ihn dem MCP-Server als Umgebungsvariable | keiner |
| `prompt_submit` (Timestamp) | Jede MCP-Tool-Antwort traegt den Timestamp; zusaetzlich ein `now`-Tool | Timestamp pro Tool-Aufruf statt pro Nachricht |
| `pre_tool` (Guards) | Guards in die Tool-Grenze verlagern (siehe unten) | Roher Shell-Zugriff bleibt ungeschuetzt, wenn der Host kein Veto bietet |
| `pre_compact` | Intervall-Extraktion (`incremental-extraction.md`: "alle ~15 Tool-Aufrufe"), der MCP-Server zaehlt selbst mit | Schwaechstes Glied: nur Claude Code hat das Event nativ |

### Guards in die Tool-Grenze

Heute prueft der Host per PreToolUse, ob der Agent loeschen oder mailen darf. Wenn die
Faehigkeit gar nicht als Tool angeboten wird, braucht es keinen Host, der sie verbietet:

- Kein `delete`-Tool, nur `archive`. Delete-Block wird zur Tool-Definition.
- Das Mail-Tool verlangt das Approval-Token im Tool selbst. Email-Guard wird zur
  Tool-Definition.

Policy wandert vom Hook in die Tool-Grenze, und die gehoert dem System. Was bleibt, ist
der rohe Shell-Zugriff des Hosts; dafuer sind die Sandbox- und Permission-Modelle des
jeweiligen Hosts zustaendig.

## Beispiel: Zwei Modelle, zwei Hosts, eine Wissensschicht

```
  claude ---- Claude Code ----+
                              +--> MCP-Server (Search, Recall, Remember, Gates)
  astra  ---- Codex CLI ------+    Wrapper (session_start, stop, Scope per CWD)
```

Jedes Modell laeuft in dem Host, der fuer es gebaut wurde. Der Switch ist eine Zeile im
Wrapper. Beide Hosts sehen denselben Hot-Memory-Stand, dieselbe Suche, dieselben offenen
Gate-4-Konflikte. Was eine Session im einen Host destilliert, hat die naechste im anderen
Host im Kontext.

Nebeneffekte:

- **Gate 7 wird real.** Zwei Frontier-Modelle mit unterschiedlichen Staerken sind das
  Panel, das `validation-layer.md` als offene Designfrage fuehrt. Fuer Gate 7 selbst
  braucht es keinen Host; das sind parallele API-Calls mit frischer Synthese-Session.
- **Der Auto-Router bekommt ein Kriterium dazu.** Neben Komplexitaet, Sensitivitaet,
  Kosten und Geschwindigkeit (`multi-llm-orchestration.md`): Aufgabentyp. Terminal-lastig
  in den einen Host, Analyse in den anderen.

## Orchestrierung ueber dem Host (optional)

Fuer Ablaeufe, die laenger als eine Session dauern oder unbeaufsichtigt laufen sollen
(etwa der Verbesserungszyklus aus `harness-evolution.md`), kann eine Runtime wie Harbor
(Go, SQLite-Backend, Event Sourcing, ein Pause/Resume-Primitiv fuer Freigaben,
Governance-Ceilings fuer Kosten) ueber den Hosts sitzen. Sie orchestriert; die
CLI-Hosts arbeiten als Tools, die sie per Shell-Out startet. Der MCP-Server steckt
direkt an.

Das ist nicht Schritt 1. Es setzt MCP-Server und einen ersten manuell gefahrenen Zyklus
voraus.

## Designentscheidungen

1. **Pull zuerst.** Der MCP-Server ist ohne Host-Wechsel nuetzlich und Voraussetzung fuer
   alles Weitere.
2. **Der Vertrag gehoert dem System, nicht dem Host.** Claude Code ist eine
   Implementierung des Adapter-Contracts, nicht seine Definition.
3. **Verluste benennen.** Timestamp pro Nachricht und PreCompact degradieren ausserhalb
   von Claude Code. Das gehoert ins README, so wie heute die Kopplung.
4. **Kein eigener Agent-Loop.** Der Host-Harness ist nicht das Kernprodukt; die
   Wissensschicht ist es. Ein eigener Loop wuerde die Harness-Qualitaet der
   Coding-Agenten aufgeben, um Portabilitaet zu gewinnen, die sich guenstiger ueber
   MCP und Adapter erreichen laesst.

## Status

Konzept, September 2026. Nichts davon ist implementiert. Reihenfolge: MCP-Server,
dann Adapter-Contract, Loesung A parallel als Sofortmassnahme fuer den Modelltausch.

---
Erstellt: 20.09.2026 | Status: Konzept
