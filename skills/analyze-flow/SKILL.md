---
name: analyze-flow
description: >
  Analyze one or more Flowcrafter Flow classes for structural issues,
  FlowBuilder validation violations, and improvement opportunities.
  Use when the user asks to "analyze a flow", "review a flow",
  "check my flow", "validate the flow", "find issues in the flow",
  "audit the workflow", or wants a code review of a *Flow.php file.
argument-hint: <flow-class-or-file> [--all]
allowed-tools: Read, Glob, Grep
---

# Analyze Flowcrafter Flow

Führe eine statische Analyse eines bestehenden Flowcrafter-Flows durch. Prüfe Korrektheit, Validierungsregel-Konformität und Verbesserungsmöglichkeiten.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als Klassenname (z.B. `WeatherComfortFlow`) oder Dateipfad. Bei `--all`: alle Flows im Projekt analysieren. Falls kein Argument: User fragen welchen Flow er analysieren möchte.

## Schritt 1: Flow lokalisieren

1. Glob auf `src/**/*Flow.php` — alle Flows finden
2. Falls Klassenname angegeben: Grep nach `class {ClassName}` um Dateipfad zu finden
3. Flow-Datei lesen

## Schritt 2: Flow-Schema parsen

Aus der `schema()`-Methode extrahieren:

1. **Type-String**: erstes Argument von `FlowBuilder`
2. **Init-Message-Klasse**: zweites Argument
3. **Return-Message-Klasse**: drittes Argument (falls vorhanden)
4. **Alle Steps**: alle `addStep()`-Aufrufe

Für jeden referenzierten Step:
1. Step-Datei lokalisieren (Glob + Grep)
2. Step lesen und extrahieren:
   - Constructor-Parameter und deren Typen
   - Message-Parameter (Types die AbstractMessage extenden oder Message-Interface implementieren)
   - Service-Parameter (alle anderen)
   - `returnTypes()`-Array
   - `process()` Return-Type-Deklaration

## Schritt 3: Validierungsregeln prüfen

### [1] Type-String-Format
- Prüfe: matcht `/^flow\..+\.v\d+$/`?
- Flag: falsches Format, fehlende Version, Leerzeichen

### [2] Init-Message konsumiert
- Prüfe: mindestens ein Step hat die Init-Message als Constructor-Parameter?
- Flag: kein Step konsumiert die Init-Message

### [3] Keine Duplikate
- Prüfe: kein Step-FQCN taucht zweimal in `addStep()` auf

### [4] Alle Steps erreichbar (BFS)
- Baue Dependency-Graph: Steps sind Knoten, Messages sind Kanten
- Starte BFS vom Step der Init-Message konsumiert
- Flag: Steps die nicht erreicht werden

### [5] Kein Zyklus (DFS)
- Führe DFS auf dem Graph durch (white/gray/black)
- Flag: erkannte Zyklen mit Pfad-Angabe

### [6] MessageDataInterface abgedeckt
- Für jeden Step: `returnTypes()` durchgehen
- Für jeden Return-Type der `MessageDataInterface` implementiert (aber NICHT `MessageReturnInterface`):
  - Prüfe: hat ein anderer Step diesen Type als Constructor-Parameter?
- Flag: nicht konsumierte `MessageDataInterface`-Types

### [7] Return-Message valide
- Falls Return-Message deklariert: muss in `returnTypes()` eines Steps erscheinen
- Die Klasse muss `MessageReturnInterface` implementieren

## Schritt 4: Verbesserungsvorschläge

1. **Fehlende `readonly`**: Sind Step-Klassen und Messages ohne `readonly`? (PHP 8.2+)
2. **Typsicherheit**: Gibt es `mixed`, fehlende Return-Types, oder fehlende Parameter-Types?
3. **Single Responsibility**: Steps mit langen `process()`-Methoden oder mehrdeutigem Namen
4. **Konsistenz**: Stimmen `returnTypes()` und `process()` Return-Type überein?
5. **Naming**: Folgen Klassen den `*Message`, `*Step`, `*Flow` Suffixen?
6. **Version**: Stimmt die Version im Type-String mit der tatsächlichen Anzahl Breaking Changes überein?

## Schritt 5: Report ausgeben

```
## Flow: {FlowClassName}
Type: flow.name.v1
Init: {InitMessageClass}
Return: {ReturnMessageClass}
Steps: N

### Validierung
[PASS/FAIL] Type-String-Format
[PASS/FAIL] Init-Message konsumiert
[PASS/FAIL] Keine Duplikate
[PASS/FAIL] Alle Steps erreichbar
[PASS/FAIL] Kein Zyklus
[PASS/FAIL] MessageDataInterface abgedeckt
[PASS/FAIL] Return-Message valide

### Gefundene Probleme
- (Liste jede Verletzung mit Erklärung und konkretem Fix-Vorschlag)

### Verbesserungsvorschläge
- (Liste jede Empfehlung)

### Message-Flow
InitMessage → StepA → DataMessage → StepB → ReturnMessage
(ASCII-Diagramm der Message-Kette)
```

## Extended Checklist

Für eine tiefere Analyse siehe `references/analysis-checklist.md`.
