---
name: create-schedule
description: >
  Generate a Flowcrafter Schedule class. Use when the user asks to
  "create a schedule", "add a schedule", "schedule a flow", "trigger flow periodically",
  "cron flow", "add FlowSchedule", or wants to run a Flowcrafter (wundii/flowcrafter)
  flow on a recurring basis via cron expression.
argument-hint: <schedule-name> <flow-class> [cron-expression]
allowed-tools: Read, Glob, Grep, Write
---

# Create Flowcrafter Schedule

Generiere eine Schedule-Klasse die einen Flowcrafter-Flow per Cron-Ausdruck triggert.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<schedule-name>` (Pflicht), `<flow-class>` (FQCN oder Kurzname des Flows, Pflicht), `<cron-expression>` (optional — frage nach falls nicht angegeben). `name` aus dem Schedule-Namen ableiten (kebab-case); `group` nur setzen, wenn das Projekt Gruppen nutzt oder der User es wünscht.

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Schedule.php` — bestehende Schedules für Namespace und Directory
2. Eine bestehende Schedule lesen um das Pattern zu bestätigen:
   - `#[FlowSchedule]`-Attribut-Verwendung (positional oder named parameters?)
   - Verwendet `$this->enqueue()` (async) oder `$this->run()` (sync)?
3. Referenzierten Flow lokalisieren (Glob auf `*{FlowClass}.php`) und die Init-Message aus `new FlowBuilder(...)` ermitteln. Ist es `EmptyInitMessage`, wird `new EmptyInitMessage()` übergeben

## Schritt 2: FlowSchedule Attribut

```php
#[FlowSchedule(
    expression: '{cron-expression}',
    name: '{human-readable-name}',
    group: '{group-name}',       // optional — für UI-Gruppierung
    active: true,                 // false = Schedule wird nicht ausgeführt
)]
```

| Parameter | Typ | Beschreibung |
|---|---|---|
| `expression` | string | Cron-Ausdruck (Pflicht) |
| `name` | string\|null | Lesbare Bezeichnung für Logs/UI |
| `group` | string\|null | Logische Gruppe im Dashboard |
| `active` | bool | Aktiviert/deaktiviert; default `true` |

### Häufige Cron-Ausdrücke

| Ausdruck | Bedeutung |
|---|---|
| `'* * * * *'` | Jede Minute (kleinste Auflösung) |
| `'*/5 * * * *'` | Alle 5 Minuten |
| `'0 * * * *'` | Jede Stunde |
| `'0 0 * * *'` | Täglich um Mitternacht |
| `'0 9 * * 1-5'` | Werktags um 9 Uhr |
| `'0 0 * * 0'` | Jeden Sonntag um Mitternacht |
| `'0 0 1 * *'` | Ersten Tag des Monats |

## Schritt 3: enqueue() vs run()

- `$this->enqueue()` — Async: legt den Flow in die Queue, der Observer verarbeitet ihn. **Empfohlen** — der Scheduler-Tick bleibt kurz
- `$this->run()` — Sync: führt den Flow direkt im Scheduler-Prozess aus, blockiert bis Abschluss. Nur für sehr kurze Flows

```php
$this->enqueue(
    flowSource: WeatherComfortFlow::class,  // FQCN des FlowInterface (Pflicht)
    message: new CityRequestMessage('...'), // Init-Message des Flows (Pflicht)
    flowSubject: 'optional-subject',        // optionaler Business-Key
    // flowHash: null,                      // bestehende Instanz erneut ausführen
    // includeSteps: [],                    // nur enqueue(): Teilausführung
);
```

## Schritt 4: Service-Abhängigkeiten

Schedules werden über den Flowcrafter-Container instanziiert. Braucht der Schedule Services (z.B. ein Repository, um pro Datensatz einen Flow zu starten), diese als `private readonly` Constructor-Parameter deklarieren und dem User nennen — sie müssen in der `DependencyRegistry` (`flowcrafter.php`) registriert sein.

## Schritt 5: Schedule-Klasse generieren

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Schedules;

use App\Flowcrafter\Flows\{FlowClass};
use App\Flowcrafter\Messages\{InitMessage};
use Wundii\Flowcrafter\Attribute\FlowSchedule;
use Wundii\Flowcrafter\Schedule\AbstractSchedule;

#[FlowSchedule('{cron-expression}', name: '{name}')]
class {ClassName}Schedule extends AbstractSchedule
{
    public function process(): void
    {
        $this->enqueue(
            flowSource: {FlowClass}::class,
            message: new {InitMessage}(/* properties */),
            // flowSubject: 'optional-subject',
        );
    }
}
```

## Schritt 6: Output

1. Generierten Code anzeigen, Cron-Ausdruck in Worten erklären
2. Prüfen ob referenzierter Flow und seine Init-Message existieren
3. Datei schreiben mit dem Write-Tool
4. Hinweise:
   - Schedules werden automatisch entdeckt — ein laufender `vendor/bin/flowcrafter scheduler` muss aber neu gestartet werden (im `dev`-Modus automatisch)
   - Bei `enqueue()` muss ein Observer (`vendor/bin/flowcrafter observer`) laufen
   - `active: false` setzen, wenn der Schedule während der Entwicklung nicht feuern soll
