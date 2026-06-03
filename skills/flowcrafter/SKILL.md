---
name: flowcrafter
description: >
  This skill should be used when the user discusses Flowcrafter, mentions
  "FlowBuilder", "StepInterface", "FlowInterface", "MessageInitInterface",
  "MessageDataInterface", "MessageReturnInterface", "AbstractMessage",
  "AbstractSchedule", "FlowSchedule", "FlowEphemeral", "FlowRunner", "FlowObserver",
  "FlowProjection", "ProjectionHandlerInterface", "ProjectionWorker",
  asks to "create a flow", "add a step", "wire up messages",
  "build a workflow", "build a processing pipeline", "define a schedule",
  "build a read model", "add a projection",
  or when PHP files named *Flow.php, *Step.php, *Message.php,
  *Schedule.php, or *Projection.php are being discussed. Also trigger when the user
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
| `MessageDataInterface` | `Wundii\Flowcrafter\Interface\MessageDataInterface` | Zwischendaten zwischen Steps. Auf dem Hauptpfad zur ReturnMessage muss jede DataMessage konsumiert werden. Auf Seitenzweigen kann sie terminal enden. |
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

**Bei komplexen Messages** (viele Werte, verschachtelte DTOs): `wundii/data-mapper` verwenden — befüllt DataMessages inkl. typisierter DTOs direkt aus HTTP-Responses (JSON/Array/XML).

**DTOs in Messages müssen `JsonSerializable` implementieren.** `AbstractMessage::jsonSerialize()` serialisiert alle promoted Properties via Reflection. Skalare Werte und Arrays werden automatisch korrekt serialisiert. Objekte (DTOs) dagegen brauchen `JsonSerializable`, da `json_encode` sonst leere Objekte oder Fehler produziert.

### Steps

Zustandslose Verarbeitungseinheiten. Jeder Step:

- Implementiert `Wundii\Flowcrafter\Interface\StepInterface`
- Deklariert verbrauchte Messages als typisierte Constructor-Parameter (auto-injected)
- Deklariert Service-Abhängigkeiten als weitere Constructor-Parameter (per DI-Container)
- Gibt mögliche Return-Types in `returnTypes(): array` als FQCN-Array an
- Implementiert `process()` mit der eigentlichen Logik

**Step Return-Types:**

| Return-Type              | Verhalten                                                                                      |
|--------------------------|------------------------------------------------------------------------------------------------|
| `MessageDataInterface`   | Message wird im Flow abgelegt, der Runner triggert rekursiv alle Steps die diese Message konsumieren |
| `MessageReturnInterface` | Flow-Ergebnis — nur der erste Return zählt, weitere werden ignoriert                           |
| `bool`                   | Wird als `FlowResult` gespeichert, keine weitere Rekursion — Step-Zweig endet hier             |

```php
class FetchWeatherStep implements StepInterface
{
    public function __construct(
        private readonly CityRequestMessage $cityRequestMessage,
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

### Step-Design-Leitfaden

**Ein Step = eine Aufgabe.** Daten holen, Daten aufbereiten und per ntfy versenden sind drei Steps, nicht einer.

**Vorgehen beim Planen:**

1. Aufgaben identifizieren und linear als Steps auflisten
2. Pro Step fragen: *Ist das Ergebnis relevant für die ReturnMessage?*
   - **Ja** → Hauptkette (Step produziert DataMessage die downstream konsumiert wird)
   - **Nein** → Seitenzweig (Step konsumiert eine Message aus der Hauptkette, Ergebnis endet terminal)

**Return-Type für Seitenzweige:**

| Situation                                        | Return-Type      |
|--------------------------------------------------|------------------|
| Simpel, selbsterklärend (Datei auf Blob legen)   | `bool`           |
| Ergebnis soll inspizierbar sein (Datenquelle befüllt → sehen was geschrieben wurde) | Terminale Message |
| Benachrichtigung (ntfy, E-Mail)                  | Judgment-Call — `bool` oder terminale Message je nach Debug-Bedarf |

**Convergence-Pattern:**

Wenn Daten aus verschiedenen Aufbereitungen zusammengeführt werden müssen, konsumiert ein nachgelagerter Step mehrere Messages:

```
m1 → s1 → m2 → s2 (externer Service) → m3 ─┐
              → s3 (interne Logik)     → m4 ─┤
                                              └→ s4(m3+m4) → m5 (Return)
```

s4 wird erst ausgeführt wenn beide Aufbereitungen (m3, m4) abgeschlossen sind.

### Flows

Workflow-Blueprints. Ein Flow:

- Implementiert `Wundii\Flowcrafter\Interface\FlowInterface`
- Deklariert `schema(): FlowSchema` via `FlowBuilder`-DSL
- Hat einen Type-String im Format `flow.<name>.v<N>` (Pflicht!)
- Verbindet Init-Message → Steps → Return-Message
- **Ein Flow = ein fachliches Ziel.** Kann viele Steps haben, aber nicht mehrere unabhängige Fachlichkeiten mischen.

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

        $flowBuilder->addStep(FetchWeatherStep::class, retries: 3, delay: 500);
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

### Projections (Read Models)

Asynchrone, entkoppelte Verarbeitung der Messages eines Flows — für Read Models, Notifications, Side-Effects. Ein Projection-Handler:

- Implementiert `Wundii\Flowcrafter\Interface\ProjectionHandlerInterface`
- Ist mit `#[Wundii\Flowcrafter\Attribute\FlowProjection(flowTypes)]` dekoriert (ein oder mehrere Flow-Type-Strings; pro Flow-Typ genau **ein** Handler)
- Bindet Methoden per `#[Wundii\Flowcrafter\Attribute\FlowProjectionMessage(MessageSource::class)]` an Message-Sources
- Jede Methode bekommt eine `Wundii\Flowcrafter\FlowMessageReadonly` (die Original-Message wird **nicht** instanziiert)
- Daten-Zugriff: `$flowMessage->getMessage()->getRawData()` liefert `array<string, mixed>`; Metadaten via `getFlowHash()`, `getFlowType()`, `getMessageSource()`, `getTime()`

```php
#[FlowProjection('flow.order.v1')]
class OrderProjection implements ProjectionHandlerInterface
{
    #[FlowProjectionMessage(OrderValidatedMessage::class)]
    public function onValidated(FlowMessageReadonly $flowMessage): void
    {
        $data = $flowMessage->getMessage()->getRawData();
        // Read Model idempotent upserten (Queue ist at-least-once)
    }
}
```

Der `FlowRunner` schreibt jede finalisierte `FlowMessage` inkrementell in die Projection-Queue — aber nur, wenn ein Handler den Flow-Typ abonniert. Der `ProjectionWorker` (`vendor/bin/flowcrafter projection:worker`) arbeitet die Queue ab. **At-least-once**: Handler-Methoden müssen idempotent sein; eine geworfene Exception wird als `ProjectionException` persistiert und die Message dennoch acked. Handler werden automatisch aus dem Composer-Classmap entdeckt.

## FlowBuilder DSL

```php
$flowBuilder = new FlowBuilder(
    'flow.name.v1',          // Type-String — muss /^flow\..+\.v\d+$/ matchen
    InitMessage::class,      // MessageInitInterface FQCN
    ReturnMessage::class,    // MessageReturnInterface FQCN (optional)
);
$flowBuilder->addStep(StepA::class);
$flowBuilder->addStep(StepB::class, retries: 3, delay: 500);
return $flowBuilder->build();
```

### `addStep()` Parameter

| Parameter | Typ | Default | Beschreibung |
|---|---|---|---|
| `stepSource` | `class-string` | — | Step-Klasse (Pflicht) |
| `retries` | `int` | `0` | Zusätzliche Versuche bei Fehler (0 = kein Retry) |
| `delay` | `int` | `200` | Wartezeit in ms zwischen Retries |

`retries` und `delay` fließen in den Schema-Hash ein — eine Änderung erzwingt eine neue Flow-Version.

## FlowBuilder Validierungsregeln (bei `build()`)

1. **Type-String-Format**: muss `/^flow\..+\.v\d+$/` matchen
2. **Init-Message konsumiert**: mindestens ein Step muss die Init-Message als Constructor-Parameter haben
3. **Keine Duplikate**: dieselbe Step-Klasse darf nicht zweimal per `addStep()` hinzugefügt werden
4. **Kein Zyklus**: kein Zyklus im Message-Dependency-Graph
5. **Alle Steps erreichbar**: jeder Step muss vom Init-Step aus über Message-Ketten erreichbar sein
6. **Hauptpfad zur ReturnMessage intakt**: Es muss mindestens ein vollständiger Pfad von InitMessage zur ReturnMessage existieren — jede DataMessage auf diesem Pfad muss downstream konsumiert werden. DataMessages auf Seitenzweigen dürfen terminal enden (nicht konsumiert, bool-Return)

## Flow-Attribute

- **`#[FlowEphemeral(expiryDays: 14)]`** — Flow ohne Primary-Storage-Persistierung, nur SQLite. Für Health-Checks, Monitoring, temporäre Flows. Details: `references/framework-concepts.md`
- **`#[FlowGroup('name')]`** — Gruppiert Flows im Dashboard. Details: `references/framework-concepts.md`

Beide nur hinzufügen wenn der User es wünscht oder das Projekt sie bereits verwendet.

## Code-Defaults

Beim Generieren von Flowcrafter-Code immer:

- `declare(strict_types=1)` setzen
- Messages: `readonly class` mit Constructor Property Promotion, Properties **`public`** (kein Getter)
- Steps: reguläre Klasse, `private readonly` Constructor-Parameter
- Flows: reguläre Klasse mit `public static function schema(): FlowSchema`
- Schedules: reguläre Klasse mit `public function process(): void`
- Alle Parameter und Return-Types explizit typisieren
- Suffixe: `*Message`, `*Step`, `*Flow`, `*Schedule`

## Directory-Conventions

Symfony: `src/Flowcrafter/{Flows,Steps,Messages,Schedules}/` — Pure PHP: `src/{Flow,Step,Message,Schedule}/`
Immer Glob verwenden um die tatsächliche Konvention im Projekt zu bestätigen!

## Framework-Detection

Vor der Code-Generierung in einem unbekannten Projekt:

1. `composer.json` lesen — Flowcrafter-Paket bestätigen, PHP-Version prüfen
2. `composer.json` auf `symfony/framework-bundle` prüfen → Symfony-Projekt
3. Glob auf `src/**/*Flow.php` → bestehende Flows und Namespace-Muster
4. Einen bestehenden Step und eine Message lesen → Namespace-Prefix und Code-Stil bestätigen

## Weiterführende Details

- `references/execution-model.md` — Rekursive Ausführung, Branching, Convergence, Regeln
- `references/testing.md` — FlowTestCase, runFlow(), runStep(), Assertions, Branching-Pattern
- `references/data-mapper.md` — `wundii/data-mapper` für typsicheres Mapping von JSON/Array/XML → DTOs
- `references/framework-concepts.md` — Message Routing, Lifecycle, Retry, Ephemeral, FlowGroup, EmptyInitMessage, DI, Testing, Imports
