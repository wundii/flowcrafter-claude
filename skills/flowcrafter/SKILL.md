---
name: flowcrafter
description: >
  This skill should be used when the user discusses Flowcrafter, mentions
  "FlowBuilder", "StepInterface", "FlowInterface", "MessageInitInterface",
  "MessageDataInterface", "MessageReturnInterface", "AbstractMessage",
  "AbstractSchedule", "FlowSchedule", "FlowRunner", "FlowObserver",
  asks to "create a flow", "add a step", "wire up messages",
  "build a workflow", "build a processing pipeline", "define a schedule",
  or when PHP files named *Flow.php, *Step.php, *Message.php,
  or *Schedule.php are being discussed. Also trigger when the user
  imports from "Wundii\Flowcrafter\" namespace.
version: 1.0.0
---

# Flowcrafter Framework Context

Flowcrafter (`wundii/flowcrafter`) ist ein PHP-Message-Driven-Workflow-Engine. Wende dieses Wissen auf alle Code-Generierungs-, Review- und Analyse-Aufgaben in einem Flowcrafter-Projekt an.

## Core-Konzepte

### Messages

Typisierte, immutable Datenobjekte. Drei Rollen:

| Interface | Namespace | Zweck |
|---|---|---|
| `MessageInitInterface` | `Wundii\Flowcrafter\Interface\MessageInitInterface` | Eintrittspunkt des Flows. Auslöser-Input. |
| `MessageDataInterface` | `Wundii\Flowcrafter\Interface\MessageDataInterface` | Zwischendaten zwischen Steps. Muss downstream konsumiert werden. |
| `MessageReturnInterface` | `Wundii\Flowcrafter\Interface\MessageReturnInterface` | Terminaler Output des Flows. Wird an den Aufrufer zurückgegeben. |

Alle Messages extends `Wundii\Flowcrafter\AbstractMessage` und sind `readonly class` mit Constructor Property Promotion.

Properties sind standardmäßig `public` — kein Getter nötig:

```php
readonly class CityRequestMessage extends AbstractMessage implements MessageInitInterface
{
    public function __construct(
        public string $city,
    ) {}
}
```

Nur auf `private` + Getter wechseln wenn der User es explizit wünscht.

### Steps

Zustandslose Verarbeitungseinheiten. Jeder Step:

- Implementiert `Wundii\Flowcrafter\Interface\StepInterface`
- Deklariert verbrauchte Messages als typisierte Constructor-Parameter
- Deklariert Service-Abhängigkeiten als weitere Constructor-Parameter (`private readonly`)
- Gibt mögliche Return-Types in `returnTypes(): array` als FQCN-Array an
- Implementiert `process()` mit der eigentlichen Logik

Flowcrafter erkennt per Reflection welche Messages ein Step benötigt. Constructor-Parameter mit Message-Typen → auto-injected. Andere Typen → per DI-Container.

```php
class FetchWeatherStep implements StepInterface
{
    public function __construct(
        private readonly CityRequestMessage $cityRequestMessage,
        // private readonly SomeService $service, // nicht-Message → DI
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [RawWeatherMessage::class];
    }

    public function process(): MessageDataInterface
    {
        return new RawWeatherMessage(/* ... */);
    }
}
```

### Flows

Workflow-Blueprints. Ein Flow:

- Implementiert `Wundii\Flowcrafter\Interface\FlowInterface`
- Deklariert `schema(): FlowSchema` via `FlowBuilder`-DSL
- Hat einen Type-String im Format `flow.<name>.v<N>` (Pflicht!)
- Verbindet Init-Message → Steps → Return-Message

```php
class WeatherComfortFlow implements FlowInterface
{
    public static function schema(): FlowSchema
    {
        $flowBuilder = new FlowBuilder(
            'flow.weather-comfort.v2',
            CityRequestMessage::class,
            WeatherReportMessage::class,
        );

        $flowBuilder->addStep(FetchWeatherStep::class);
        $flowBuilder->addStep(ConvertWeatherStep::class);
        $flowBuilder->addStep(SummaryReportStep::class);

        return $flowBuilder->build();
    }
}
```

### Schedules

Cron-gesteuerte Flow-Starter. Ein Schedule:

- Extends `Wundii\Flowcrafter\Schedule\AbstractSchedule`
- Ist mit `#[Wundii\Flowcrafter\Attribute\FlowSchedule]` dekoriert
- Implementiert `process(): void`
- Verwendet `$this->enqueue()` (async) oder `$this->run()` (sync)

```php
#[FlowSchedule('* * * * *', name: 'weather-comfort-schedule')]
class WeatherComfortSchedule extends AbstractSchedule
{
    public function process(): void
    {
        $this->enqueue(
            flowSource: WeatherComfortFlow::class,
            message: new CityRequestMessage('Tokyo'),
            flowSubject: 'tokyo',
        );
    }
}
```

## FlowBuilder DSL

```php
$flowBuilder = new FlowBuilder(
    'flow.name.v1',          // Type-String — muss /^flow\..+\.v\d+$/ matchen
    InitMessage::class,      // MessageInitInterface FQCN
    ReturnMessage::class,    // MessageReturnInterface FQCN (optional)
);
$flowBuilder->addStep(StepA::class);
$flowBuilder->addStep(StepB::class);
return $flowBuilder->build();
```

## FlowBuilder Validierungsregeln (bei `build()`)

1. **Type-String-Format**: muss `/^flow\..+\.v\d+$/` matchen
2. **Init-Message konsumiert**: mindestens ein Step muss die Init-Message als Constructor-Parameter haben
3. **Keine Duplikate**: dieselbe Step-Klasse darf nicht zweimal per `addStep()` hinzugefügt werden
4. **Kein Zyklus**: kein Zyklus im Message-Dependency-Graph
5. **Alle Steps erreichbar**: jeder Step muss vom Init-Step aus über Message-Ketten erreichbar sein
6. **MessageDataInterface abgedeckt**: jeder `MessageDataInterface`-Return-Type muss downstream konsumiert werden

## FlowGroup — Flows im Dashboard gruppieren

Das optionale `#[FlowGroup]`-Attribut (`Wundii\Flowcrafter\Attribute\FlowGroup`) gruppiert Flows im Flowcrafter-Dashboard:

```php
use Wundii\Flowcrafter\Attribute\FlowGroup;

#[FlowGroup('weather')]
class WeatherComfortFlow implements FlowInterface
{
    // ...
}
```

Nur hinzufügen wenn der User explizit eine Gruppierung wünscht oder das Projekt bereits `#[FlowGroup]` verwendet.

## EmptyInitMessage — Flow ohne externen Input

Falls ein Flow keinen externen Input benötigt (z.B. ein Scheduler-getriggerter Flow), **keine eigene Init-Message erstellen**. Stattdessen die eingebaute Klasse verwenden:

```php
use Wundii\Flowcrafter\EmptyInitMessage;

$flowBuilder = new FlowBuilder('flow.my-flow.v1', EmptyInitMessage::class);
```

Im ersten Step **muss** das `EmptyInitMessage`-Property `public readonly` sein (nicht `private`), damit statische Analyse-Tools (z.B. Rector) den "unbenutzten" Parameter nicht entfernen:

```php
class MyFirstStep implements StepInterface
{
    public function __construct(
        public readonly EmptyInitMessage $init,  // public readonly — Pflicht!
    ) {}
}
```

## Korrekte Imports

```php
use Wundii\Flowcrafter\AbstractMessage;
use Wundii\Flowcrafter\FlowBuilder;
use Wundii\Flowcrafter\FlowSchema;
use Wundii\Flowcrafter\Interface\FlowInterface;
use Wundii\Flowcrafter\Interface\StepInterface;
use Wundii\Flowcrafter\Interface\MessageInitInterface;
use Wundii\Flowcrafter\Interface\MessageDataInterface;
use Wundii\Flowcrafter\Interface\MessageReturnInterface;
use Wundii\Flowcrafter\EmptyInitMessage;
use Wundii\Flowcrafter\Attribute\FlowGroup;
use Wundii\Flowcrafter\Attribute\FlowSchedule;
use Wundii\Flowcrafter\Schedule\AbstractSchedule;
```

## Directory-Conventions

**Symfony-Projekte** (häufiges Layout):
```
src/
└── Flowcrafter/
    ├── Flows/          ← *Flow.php
    ├── Steps/          ← *Step.php
    ├── Messages/       ← *Message.php
    └── Schedules/      ← *Schedule.php
```

**Pure PHP**:
```
src/
├── Flow/
├── Step/
├── Message/
└── Schedule/
```

Immer Glob verwenden um die tatsächliche Konvention im Projekt zu bestätigen!

## Code-Defaults

Beim Generieren von Flowcrafter-Code immer:

- `declare(strict_types=1)` setzen
- Messages: `readonly class` mit Constructor Property Promotion, Properties **`public`** (kein Getter) — außer der User wünscht explizit `private` + Getter
- Steps: reguläre Klasse, `private readonly` Constructor-Parameter
- Flows: reguläre Klasse mit `public static function schema(): FlowSchema`
- Schedules: reguläre Klasse mit `public function process(): void`
- Alle Parameter und Return-Types explizit typisieren
- Suffixe: `*Message`, `*Step`, `*Flow`, `*Schedule`

## Framework-Detection

Vor der Code-Generierung in einem unbekannten Projekt:

1. `composer.json` lesen — Flowcrafter-Paket bestätigen, PHP-Version prüfen
2. `composer.json` auf `symfony/framework-bundle` prüfen → Symfony-Projekt
3. Glob auf `src/**/*Flow.php` → bestehende Flows und Namespace-Muster
4. Einen bestehenden Step und eine Message lesen → Namespace-Prefix und Code-Stil bestätigen

## Weiterführende Details

- `references/framework-concepts.md` — Message Routing, Execution Lifecycle, Symfony Bundle Integration
