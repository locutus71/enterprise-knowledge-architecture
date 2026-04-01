# Timestamp Hook

## Kernidee

KI-Agenten haben kein Zeitbewusstsein. Sie wissen nicht wann eine Nachricht geschrieben wurde, wie viel Zeit zwischen zwei Nachrichten vergangen ist, oder ob der User morgens oder nachts arbeitet. Der Timestamp-Hook loest das.

## Implementierung

Ein UserPromptSubmit-Hook injiziert bei jeder User-Nachricht automatisch einen Timestamp in den Kontext.

Format: `[Timestamp: Di 2026-03-24 14:53:18]`

Bestandteile:
- Wochentag (deutsch, abgekuerzt)
- Datum (ISO-Format)
- Uhrzeit (Sekunden-genau)

## Was das ermoeglicht

### Zeitabstaende berechnen
"Zwischen deiner letzten und dieser Nachricht sind 25 Minuten vergangen." Der Agent kann erkennen ob der User schnell arbeitet oder lange pausiert hat.

### Arbeitszeitmuster erkennen
Morgens produktiv, abends kreativ, nachts ueberfokussiert? Der Agent sieht die Muster ueber Sessions hinweg (wenn in Deep Storage geloggt).

### Exakte Uhrzeiten in Logs
Statt "irgendwann am Dienstag" steht im BrainDB-Log: "Di 14:53 Uhr". Praezise Dokumentation ohne manuellen Aufwand.

### Coach-Funktion
"Es ist 23:30 und du arbeitest seit 4 Stunden. Morgen ist ein wichtiger Termin." Zeitbewusste Beratung statt zeitloser Empfehlungen.

### Chronologische Rekonstruktion
Bei Context-Compaction (Kontextfenster voll, wird komprimiert) bleiben die Timestamps als Anker. Man kann rekonstruieren wann was passiert ist, auch nach Komprimierung.

## Designentscheidungen

1. **UserPromptSubmit statt SessionStart:** Nicht einmal pro Session, sondern bei JEDER Nachricht. Zeitabstaende zwischen Nachrichten sind die wertvollste Information.
2. **Deutscher Wochentag:** Der User arbeitet auf Deutsch. "Di" ist intuitiver als "Tue".
3. **Sekunden-Genauigkeit:** Ermoeglicht die Unterscheidung von schnellen Interaktionen (Sekunden) und Denkpausen (Minuten).
4. **Minimaler Overhead:** 3 Sekunden Timeout. Ein Python-Einzeiler. Kein merkbarer Impact auf die Responsiveness.

---
Erstellt: 24.03.2026 | Autor: Holger Woelfle
