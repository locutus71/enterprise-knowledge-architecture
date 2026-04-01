# Hook System

## Kernidee

Hooks sind automatisierte Trigger die bei bestimmten Events im Session-Lebenszyklus feuern. Sie machen aus einem passiven Chat ein aktives Betriebssystem. Kein manuelles Kontextladen, kein manuelles Speichern. Die Architektur erzwingt richtiges Verhalten.

## 5-Stufen Session-Lifecycle

```
SessionStart → UserPromptSubmit → PreToolUse → PreCompact → Stop
```

### 1. SessionStart

Feuert einmal bei Session-Beginn. Laedt den gesamten Kontext.

Ablauf (10 Schritte):
1. Scope erkennen (welches Projekt per CWD)
2. Statistiken der Search Engine ausgeben
3. Hot Memory Fakten injizieren (User-Kontext: Name, Rolle, OS)
4. CWD-basierter Projekt-Kontext laden
5. Fehler-Datenbank (BUGLIST) einlesen
6. Deep Storage Stats + letzte 3 Logs injizieren
7. Offene Konflikte als Warnung ausgeben
8. Tagesplan laden (Coach-Funktion)
9. Auf ~2.000 Token begrenzen (Token-Budget)
10. Search Engine im Hintergrund vorwaermen

Ergebnis: Der Agent startet nie bei null. Er weiss wer fragt, was der Stand ist, welche Fehler bekannt sind, was heute geplant ist.

### 2. UserPromptSubmit

Feuert bei jeder Nachricht des Users.

- Injiziert Timestamp mit Wochentag und Uhrzeit
- Format: `[Timestamp: Di 2026-03-24 14:53:18]`
- Ermoeglicht: Zeitabstaende berechnen, Arbeitszeitmuster erkennen, exakte Uhrzeiten in Logs

### 3. PreToolUse

Feuert bevor der Agent ein Tool ausfuehrt. Safety Layer.

**Email-Guard:** Blockiert jeden Email-Versand-Befehl ohne explizites Approval-Token. Der Agent kann keine Emails versenden ohne dass der User es freigibt. Kein Undo noetig, weil der Fehler verhindert wird.

**Delete-Block:** Verhindert das Loeschen von Dateien. Erzwingt Archivierung statt Loeschung. Policy as Code.

### 4. PreCompact

Feuert bevor das Kontextfenster komprimiert wird.

Problem: Bei langen Sessions wird der Kontext automatisch komprimiert. Dabei gehen Details verloren.
Loesung: Vor der Komprimierung wird der aktuelle Fortschritt als Log in Deep Storage geschrieben. Basiert auf dem "Focus Loop Pattern". Kein Wissensverlust durch Komprimierung.

### 5. Stop

Feuert bei Session-Ende. Knowledge Capture.

Ablauf (5 Schritte):
1. Token-Verbrauch und Kosten archivieren
2. Stats-Cache aktualisieren
3. Session-Transcript als Deep Storage Log
4. Deep Storage Notes nach Markdown exportieren (fuer Search Engine)
5. Reindex pruefen und ggf. starten (Schwellenwerte: 500k Token ODER 100 Queries ODER 5 geaenderte Dateien)

## Designprinzip: Policy as Code

Regeln in Dokumentation werden vergessen. Regeln als Hooks werden erzwungen.

- "Keine Emails ohne Freigabe" → Email-Guard Hook
- "Keine Dateien loeschen" → Delete-Block Hook
- "Kontext laden" → SessionStart Hook
- "Wissen sichern" → PreCompact + Stop Hooks

Das System braucht keine Disziplin, weil die Architektur Fehler verhindert.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
