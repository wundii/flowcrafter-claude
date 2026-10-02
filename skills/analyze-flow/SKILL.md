---
name: analyze-flow
description: >
  Analyze one or more Flowcrafter Flow classes for structural issues,
  FlowBuilder validation violations, and improvement opportunities.
  Use when the user asks to "analyze a flow", "review a flow",
  "check my flow", "validate the flow", "find issues in the flow",
  "audit the workflow", or wants a code review of a Flowcrafter *Flow.php file.
argument-hint: <flow-class-or-file> [--all]
allowed-tools: Read, Glob, Grep, Bash(php:*)
---

# Analyze Flowcrafter Flow

Führe eine Analyse eines bestehenden Flowcrafter-Flows durch. Prüfe Korrektheit, Validierungsregel-Konformität, Laufzeit-Fallstricke und Verbesserungsmöglichkeiten. **Nur analysieren, keine Dateien ändern** — Fixes als Vorschläge ausgeben.

Konzepte und Regeln: `../flowcrafter/SKILL.md`, `../flowcrafter/references/execution-model.md`, `../create-flow/references/flowbuilder-validation.md`.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als Klassenname (z.B. `WeatherComfortFlow`) oder Dateipfad. Bei `--all`: alle Flows im Projekt analysieren (bei vielen Flows erst eine Übersichtstabelle, dann Details nur zu Flows mit Befunden). Falls kein Argument: User fragen welcher Flow analysiert werden soll.

## Schritt 1: Flow lokalisieren

1. Glob auf `src/**/*Flow.php` — alle Flows finden; nur Klassen mit `implements FlowInterface` berücksichtigen
2. Falls Klassenname angegeben: Grep nach `class {ClassName}` um Dateipfad zu finden
3. Flow-Datei lesen

## Schritt 2: Echte Validierung (falls möglich)

Statische Graph-Analyse durch Lesen ist fehleranfällig. Wenn `vendor/autoload.php` existiert, das Schema vom Framework selbst bauen lassen:

```bash
php -r 'require "vendor/autoload.php"; $s = \App\Flowcrafter\Flows\{ClassName}Flow::schema(); echo $s->type(), PHP_EOL, json_encode($s, JSON_PRETTY_PRINT), PHP_EOL;'
```

- Erfolg → Regeln [1]–[8] sind **PASS**; das JSON liefert für jeden Step `messages`, `returnTypes`, `retries`, `delay`, `runOnce` — als Grundlage für Schritt 4 verwenden.
- `InvalidArgumentException` → die Meldung einer Regel zuordnen (siehe `flowbuilder-validation.md`), als **FAIL** berichten und trotzdem statisch weiteranalysieren.
- Andere Fehler (kein PHP lokal, Docker-only-Projekt, fehlende Extensions) → im Report vermerken und rein statisch analysieren.

Der `php -r`-Aufruf ist rein lesend (`schema()` ist statisch und baut nur den Graph). Keine anderen Befehle ausführen, die Dateien schreiben (z.B. `diagram:mermaid`).

## Schritt 3: Flow-Schema statisch parsen

Aus der `schema()`-Methode extrahieren:

1. **Type-String**: erstes Argument von `FlowBuilder`
2. **Init-Message-Klasse**: zweites Argument
3. **Return-Message-Klasse**: drittes Argument (falls vorhanden)
4. **Alle Steps**: alle `addStep()`-Aufrufe inkl. `retries`, `delay`, `runOnce`
5. **Attribute**: `#[FlowEphemeral]`, `#[FlowGroup]`

Für jeden referenzierten Step:
1. Step-Datei lokalisieren (Glob + Grep)
2. Step lesen und extrahieren:
   - Constructor-Parameter und deren Typen
   - Message-Parameter (Typen die `MessageInterface` implementieren)
   - Service-Parameter (alle anderen)
   - `returnTypes()`-Array (und ob es auf `$this` zugreift)
   - `process()` Return-Type-Deklaration und tatsächlich zurückgegebene Klassen
   - `extends AbstractStep`? Ruft er `enqueue()`/`run()` auf?
   - try/catch-Verhalten bei Seiteneffekten

## Schritt 4: Validierungsregeln prüfen

Wenn Schritt 2 erfolgreich war, [1]–[8] als PASS übernehmen. Sonst statisch prüfen:

- **[1] Type-String-Format** — matcht `/^flow\..+\.v\d+$/`?
- **[2] Init-Message konsumiert** — mindestens ein Step hat sie als Constructor-Parameter?
- **[3] Keine Duplikate** — kein Step-FQCN zweimal in `addStep()`?
- **[4] Alle Steps erreichbar** — BFS vom Init-Step über Messages
- **[5] Kein Zyklus** — DFS auf dem Step-Graph
- **[6] Keine hängenden DataMessages** — jeder Return-Type, der nicht `MessageReturnInterface` ist, wird von irgendeinem Step konsumiert (keine Ausnahme für Seitenzweige)
- **[7] Return-Message valide** — implementiert `MessageReturnInterface` und steht in `returnTypes()` eines Steps
- **[8] Step-Constructor/`returnTypes()`** — jeder Step hat einen Constructor; `returnTypes()` ist ein konstantes Array ohne `$this`-Zugriff

Diese Regeln prüft der FlowBuilder **nicht** — immer statisch prüfen:

- **[9] Ein Produzent pro Message-Typ** — kein Message-Typ in `returnTypes()` von zwei Steps
- **[10] Eindeutiges Flow-Ergebnis** — nur ein Step (auf der Hauptkette) liefert eine `MessageReturnInterface`; sonst gewinnt die zuerst produzierte
- **[11] `returnTypes()` ↔ `process()`** — jede Klasse, die `process()` zurückgeben kann, steht in `returnTypes()` und umgekehrt
- **[12] Fehlerisolation in Seitenzweigen** — Seiteneffekt-Steps (Notification, Logging, Cache) werfen nicht ungefangen, sonst bricht der ganze Flow ab. Bei gefangenem Fehler `false` (→ `WARNING`) statt `true`
- **[13] Sub-Flows** — Steps mit `enqueue()` erfordern eine Queue im Runner (Tests: `InMemoryQueue`); `run()`-Ketten auf Rekursion zwischen Flows prüfen
- **[14] `runOnce`** — Steps mit nicht wiederholbaren Seiteneffekten (Zahlung, Mail, externe Buchung) haben `runOnce: true`
- **[15] Projections passen zur Version** — Grep `#[FlowProjection` nach dem Flow-Namen: abonniert ein Handler nur eine alte Version (`v1` statt aktueller `v2`)?
- **[16] FlowEphemeral** — falls gesetzt: `expiryDays >= 1`, keine Business-Daten, Scheduler läuft (Cleanup)
- **[17] Type-String eindeutig** — Grep über das Projekt: kein anderer Flow mit demselben Type-String

## Schritt 5: Verbesserungsvorschläge

Siehe `references/analysis-checklist.md`. Die wichtigsten:

1. **Messages**: `readonly class`, nur promoted Properties, DTOs round-trip-fähig (Constructor-Parameternamen = serialisierte Keys)
2. **Typsicherheit**: kein `mixed`, alle Return-Types deklariert
3. **Single Responsibility**: lange `process()`-Methoden oder mehrdeutige Namen
4. **Retries**: externer I/O mit `retries`, reine Transformationen ohne
5. **DI**: alle Service-Parameter in der `DependencyRegistry` registrierbar (Interfaces per `bind()`)
6. **Naming**: `*Message`, `*Step`, `*Flow` Suffixe
7. **Tests**: existiert ein `FlowTestCase` für den Flow (Grep auf `{ClassName}::class` in `tests/`)?

## Schritt 6: Report ausgeben

```
## Flow: {FlowClassName}
Type: flow.name.v1
Init: {InitMessageClass}
Return: {ReturnMessageClass}
Steps: N
Validierung: echt (php) | statisch (Grund)

### Validierung (FlowBuilder)
[PASS/FAIL] [1] Type-String-Format
[PASS/FAIL] [2] Init-Message konsumiert
[PASS/FAIL] [3] Keine Duplikate
[PASS/FAIL] [4] Alle Steps erreichbar
[PASS/FAIL] [5] Kein Zyklus
[PASS/FAIL] [6] Keine hängenden DataMessages
[PASS/FAIL] [7] Return-Message valide
[PASS/FAIL] [8] Step-Constructor / returnTypes()

### Laufzeit-Regeln (nicht vom FlowBuilder geprüft)
[PASS/WARN] [9]  Ein Produzent pro Message-Typ
[PASS/WARN] [10] Eindeutiges Flow-Ergebnis
[PASS/WARN] [11] returnTypes() ↔ process()
[PASS/WARN] [12] Fehlerisolation in Seitenzweigen
[PASS/WARN/N/A] [13] Sub-Flows
[PASS/WARN/N/A] [14] runOnce
[PASS/WARN/N/A] [15] Projections passen zur Version
[PASS/WARN/N/A] [16] FlowEphemeral
[PASS/WARN] [17] Type-String eindeutig

### Gefundene Probleme
- (jede Verletzung mit Datei:Zeile, Erklärung und konkretem Fix-Vorschlag)

### Verbesserungsvorschläge
- (jede Empfehlung)

### Message-Flow
(ASCII-Diagramm der Message-Kette inkl. Seitenzweige und Convergence)
```
