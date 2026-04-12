---
name: create-schedule
description: >
  Generate a Flowcrafter Schedule class. Use when the user asks to
  "create a schedule", "add a schedule", "schedule a flow", "trigger flow periodically",
  "cron flow", "add FlowSchedule", or wants to run a Flowcrafter flow
  on a recurring basis via cron expression.
argument-hint: <schedule-name> <flow-class> [cron-expression]
allowed-tools: Read, Glob, Grep, Write
---

# Create Flowcrafter Schedule

Generiere eine Schedule-Klasse die einen Flowcrafter-Flow per Cron-Ausdruck triggert.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<schedule-name>` (Pflicht), `<flow-class>` (FQCN oder Kurzname des Flows, Pflicht), `<cron-expression>` (optional — frage nach falls nicht angegeben). Frage auch nach `name` und `group` für den `#[FlowSchedule]`-Attribute.

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Schedule.php` — bestehende Schedules für Namespace und Directory
2. Eine bestehende Schedule lesen um das Pattern zu bestätigen:
   - `#[FlowSchedule]`-Attribut-Verwendung (positional oder named parameters?)
   - Verwendet `$this->enqueue()` (async) oder `$this->run()` (sync)?
3. Referenzierten Flow lokalisieren (Glob auf `*{FlowClass}.php`) und Init-Message ermitteln

## Schritt 2: FlowSchedule Attribut

```php
#[FlowSchedule(
    expression: '{cron-expression}',
    name: '{human-readable-name}',
    group: '{group-name}',       // optional — für UI-Gruppierung
    active: true,                 // false während Entwicklung/Testing
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
| `'* * * * *'` | Jede Minute |
| `'*/5 * * * *'` | Alle 5 Minuten |
| `'0 * * * *'` | Jede Stunde |
| `'0 0 * * *'` | Täglich um Mitternacht |
| `'0 9 * * 1-5'` | Werktags um 9 Uhr |
| `'0 0 * * 0'` | Jeden Sonntag um Mitternacht |
| `'0 0 1 * *'` | Ersten Tag des Monats |

## Schritt 3: enqueue() vs run()

- `$this->enqueue()` — Async: legt Job in die Queue, empfohlen für produktive Schedules
- `$this->run()` — Sync: führt Flow direkt aus, blockiert bis Abschluss

Beide Methoden haben dieselbe Signatur:
```php
$this->enqueue(
    flowSource: WeatherComfortFlow::class,  // FQCN des FlowInterface
    message: new CityRequestMessage('...'), // Init-Message des Flows
    flowSubject: 'optional-subject',        // optionaler Business-Key
);
```

## Schritt 4: Schedule-Klasse generieren

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Schedules;

use App\Flowcrafter\Flows\{FlowClass};
use App\Flowcrafter\Messages\{InitMessage};
use Wundii\Flowcrafter\Attribute\FlowSchedule;
use Wundii\Flowcrafter\Schedule\AbstractSchedule;

#[FlowSchedule('{cron-expression}', name: '{name}', active: true)]
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

## Schritt 5: Output

1. Generierten Code anzeigen
2. Cron-Ausdruck bestätigen falls vom User nicht angegeben
3. Prüfen ob referenzierter Flow und seine Init-Message existieren
4. Hinweis: `active: false` während der Entwicklung/Testing setzen
5. Datei schreiben mit dem Write-Tool
