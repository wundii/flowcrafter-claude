---
name: flowcrafter
description: >
  Framework knowledge for the Flowcrafter PHP workflow engine (wundii/flowcrafter).
  Use whenever the user discusses Flowcrafter or mentions "FlowBuilder",
  "StepInterface", "FlowInterface", "MessageInitInterface", "MessageDataInterface",
  "MessageReturnInterface", "AbstractMessage", "AbstractStep", "AbstractSchedule",
  "EmptyInitMessage", "FlowSchedule", "FlowEphemeral", "FlowGroup", "FlowRunner",
  "FlowObserver", "FlowTestCase", "DependencyRegistry", "FlowProjection",
  "ProjectionHandlerInterface", "ProjectionWorker", when PHP files named *Flow.php,
  *Step.php, *Message.php, *Schedule.php or *Projection.php in a Flowcrafter project
  are being discussed or edited, or when code imports from the "Wundii\Flowcrafter\"
  namespace. Provides concepts, validation rules and code defaults used by the
  create-*/analyze-flow skills.
---

# Flowcrafter Framework Context

Flowcrafter (`wundii/flowcrafter`) ist eine PHP-Message-Driven-Workflow-Engine. Wende dieses Wissen auf alle Code-Generierungs-, Review- und Analyse-Aufgaben in einem Flowcrafter-Projekt an.

> **Source of Truth:** Das Paket liefert seine Doku mit. Falls `vendor/wundii/flowcrafter/docs/` existiert, bei Unklarheiten bzw. Detailfragen dort nachlesen (`concepts.md`, `testing.md`, `configuration.md`) — sie passt immer zur installierten Version. Bei Widerspruch zwischen diesem Skill und dem Code im `vendor/` gilt der Code.

## Core-Konzepte

### Messages

Typisierte, immutable Datenobjekte. Drei Rollen:

| Interface | Namespace | Zweck |
|---|---|---|
| `MessageInitInterface` | `Wundii\Flowcrafter\Interface\MessageInitInterface` | Eintrittspunkt des Flows. Auslöser-Input. |
| `MessageDataInterface` | `Wundii\Flowcrafter\Interface\MessageDataInterface` | Zwischendaten zwischen Steps. **Muss immer** von mindestens einem Step im Flow konsumiert werden — auch auf Seitenzweigen (sonst schlägt `build()` fehl). |
| `MessageReturnInterface` | `Wundii\Flowcrafter\Interface\MessageReturnInterface` | Terminaler Output des Flows. Wird an den Aufrufer zurückgegeben. Wird nicht weiter konsumiert. |

Alle Messages extends `Wundii\Flowcrafter\AbstractMessage`. Da `AbstractMessage` eine `abstract readonly class` ist, **müssen** Messages `readonly class` sein (PHP ≥ 8.2), mit Constructor Property Promotion.

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

**Serialisierung & Rehydrierung (Round-Trip):**

- `AbstractMessage::jsonSerialize()` serialisiert **nur promoted Constructor-Properties** (via Reflection). Nicht-promoted oder berechnete Properties gehen verloren.
- Beim Laden aus Storage/Queue (Observer, Re-Runs, `runOnce`, `includeSteps`) wird die Message per `wundii/data-mapper` mit `ApproachEnum::CONSTRUCTOR` wiederhergestellt. Die serialisierten Keys müssen also den **Constructor-Parameternamen** entsprechen.
- **DTOs in Messages** müssen denselben Round-Trip überstehen: entweder `public` promoted Properties, oder `JsonSerializable` mit Keys, die exakt den Constructor-Parameternamen des DTOs entsprechen. `DateTimeInterface` wird beim Laden standardmäßig auf `DateTime` gemappt.
- Das Umbenennen einer Property ändert den Message-Hash → ändert den Schema-Hash des Flows (siehe Versionierung).

**Bei komplexen Messages** (viele Werte, verschachtelte DTOs): `wundii/data-mapper` verwenden — befüllt DataMessages inkl. typisierter DTOs direkt aus HTTP-Responses (JSON/Array/XML). Details: `references/data-mapper.md`.

### Steps

Zustandslose Verarbeitungseinheiten. Jeder Step:

- Implementiert `Wundii\Flowcrafter\Interface\StepInterface` **oder** extends `Wundii\Flowcrafter\AbstractStep` (das `StepInterface` implementiert)
- **Hat einen Constructor** (ohne Constructor wirft `addStep()`)
- Deklariert verbrauchte Messages als typisierte Constructor-Parameter (auto-injected; erkannt wird jeder Parametertyp, der `MessageInterface` implementiert — Union-Types werden ignoriert)
- Deklariert Service-Abhängigkeiten als weitere Constructor-Parameter (per Flowcrafter-DI-Container, siehe DI)
- Gibt mögliche Return-Types in `returnTypes(): array` als FQCN-Array an — **konstantes Array**, darf nicht auf `$this`-Properties zugreifen (wird auf einer Instanz ohne Constructor-Aufruf ermittelt)
- Implementiert `process()` mit der eigentlichen Logik

**Step Return-Types:**

| Return-Type              | Verhalten                                                                                      |
|--------------------------|------------------------------------------------------------------------------------------------|
| `MessageDataInterface`   | Message wird im Flow abgelegt, der Runner triggert rekursiv alle Steps die diese Message konsumieren |
| `MessageReturnInterface` | Flow-Ergebnis — **nur die erste** produzierte Return-Message zählt, weitere werden ignoriert   |
| `bool`                   | Wird als `FlowResult` gespeichert, keine weitere Rekursion. `false` → Flow-Status `WARNING`     |

**Exceptions:** Wirft ein Step (nach Erschöpfung der Retries), bricht der **gesamte** Flow-Run ab — auch die noch nicht ausgeführten Steps der Hauptkette. Das gilt ebenso für Seitenzweige.

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

**Sub-Flows aus einem Step triggern:** Extends ein Step `AbstractStep`, stehen ihm — analog zu `AbstractSchedule` — `$this->enqueue()` (async via Queue) und `$this->run()` (synchron, blockierend) zur Verfügung:

```php
$this->enqueue(
    flowSource: ShipmentFlow::class,                           // Pflicht
    message: new ShipmentRequestMessage($this->order->orderId), // Pflicht: Init-Message des Sub-Flows
    // flowHash: null,      // bestehende Instanz erneut ausführen
    // flowSubject: null,   // Business-Key
    // includeSteps: [],    // nur bei enqueue()
);
```

- `enqueue()` braucht eine Queue im Runner — ohne wirft es `RuntimeException('Queue is not set.')` (in Tests: `InMemoryQueue`).
- `run()` hat keinen Zyklenschutz über Flow-Grenzen — rekursive Ketten können den Stack sprengen.
- `AbstractStep` liefert zusätzlich `getFlowHash()`, `getFlowRuntimeHash()`, `getFlowType()`, `getFlowSchemaHash()`, `getFlowSubject()`.
- `StepInterface` direkt zu implementieren bleibt der Default — `AbstractStep` nur, wenn der Step Flow-Metadaten oder Sub-Flows braucht.

### Step-Design-Leitfaden

**Ein Step = eine Aufgabe.** Daten holen, Daten aufbereiten und per ntfy versenden sind drei Steps, nicht einer.

**Vorgehen beim Planen:**

1. Aufgaben identifizieren und linear als Steps auflisten
2. Pro Step fragen: *Ist das Ergebnis relevant für die ReturnMessage?*
   - **Ja** → Hauptkette (Step produziert DataMessage die downstream konsumiert wird)
   - **Nein** → Seitenzweig (Step konsumiert eine Message aus der Hauptkette und endet mit `bool`)

**Seitenzweige enden mit `bool`.** Eine DataMessage als Ende ist nicht erlaubt (unkonsumiert → `build()` schlägt fehl). Eine zweite `MessageReturnInterface` als Ende ist technisch möglich, aber gefährlich: läuft der Seitenzweig vor der Hauptkette, wird **seine** Message zum Flow-Ergebnis („erste Return-Message gewinnt“). Soll das Ergebnis eines Seitenzweigs inspizierbar sein, die Details per Ausgabe/Logging oder Projection sichtbar machen — nicht per Return-Message.

| Situation                                        | Return                                       |
|--------------------------------------------------|----------------------------------------------|
| Seiteneffekt erfolgreich / nicht nötig           | `true`                                       |
| Seiteneffekt fehlgeschlagen, Flow soll weiterlaufen | Exception im Step fangen → `false` (`WARNING`) |
| Fehler soll den Flow abbrechen                   | Exception werfen (`FAILED`)                  |
| Seiteneffekt darf bei Re-Runs nicht wiederholt werden (Zahlung, Mail) | `addStep(..., runOnce: true)` |

**Convergence-Pattern:**

Wenn Daten aus verschiedenen Aufbereitungen zusammengeführt werden müssen, konsumiert ein nachgelagerter Step mehrere Messages:

```
m1 → s1 → m2 → s2 (externer Service) → m3 ─┐
              → s3 (interne Logik)     → m4 ─┤
                                              └→ s4(m3+m4) → m5 (Return)
```

s4 wird erst ausgeführt wenn beide Aufbereitungen (m3, m4) abgeschlossen sind.

**Ein Message-Typ = ein Produzent.** Zwei Steps im selben Flow dürfen nicht denselben Message-Typ produzieren. `FlowBuilder` prüft das **nicht** — es muss beim Entwurf (und in `analyze-flow`) sichergestellt werden.

### Flows

Workflow-Blueprints. Ein Flow:

- Implementiert `Wundii\Flowcrafter\Interface\FlowInterface`
- Deklariert `public static function schema(): FlowSchema` via `FlowBuilder`-DSL
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
- Verwendet `$this->enqueue()` (async, empfohlen) oder `$this->run()` (sync) — gleiche Signatur wie bei `AbstractStep`

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
- Bindet `public` Methoden per `#[Wundii\Flowcrafter\Attribute\FlowProjectionMessage(MessageSource::class)]` an Message-Sources
- Jede Methode bekommt eine `Wundii\Flowcrafter\FlowMessageReadonly` (die Original-Message wird **nicht** instanziiert)
- Daten-Zugriff: `$flowMessage->getMessage()->getRawData()` liefert `array<string, mixed>`; Metadaten via `getHash()` (eindeutig pro Message, für Idempotenz), `getFlowHash()`, `getFlowType()`, `getMessageSource()`, `getTime()`

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

Der `FlowRunner` schreibt jede finalisierte `FlowMessage` inkrementell in die Projection-Queue — aber nur, wenn ein Handler den Flow-Typ abonniert. Der `ProjectionWorker` (`vendor/bin/flowcrafter projection:worker`) arbeitet die Queue ab. **At-least-once**: Handler-Methoden müssen idempotent sein; eine geworfene Exception wird als `ProjectionException` persistiert und die Message dennoch acked. Handler werden pro Worker-Prozess einmal gebaut — keinen Zustand zwischen Messages halten.

**Achtung bei Versions-Bump:** `#[FlowProjection]` abonniert exakte Type-Strings. Wird `flow.order.v1` → `v2`, muss der neue Typ im Attribut ergänzt werden — sonst wird der neue Flow stillschweigend nicht mehr projiziert.

## FlowBuilder DSL

```php
$flowBuilder = new FlowBuilder(
    'flow.name.v1',          // Type-String — muss /^flow\..+\.v\d+$/ matchen
    InitMessage::class,      // MessageInitInterface FQCN
    ReturnMessage::class,    // MessageReturnInterface FQCN (optional)
);
$flowBuilder->addStep(StepA::class);
$flowBuilder->addStep(StepB::class, retries: 3, delay: 500);
$flowBuilder->addStep(ChargeStep::class, runOnce: true);
return $flowBuilder->build();
```

### `addStep()` Parameter

| Parameter | Typ | Default | Beschreibung |
|---|---|---|---|
| `step` | `class-string<StepInterface>` | — | Step-Klasse (Pflicht, positional übergeben) |
| `retries` | `int` | `0` | Zusätzliche Versuche bei Exception (0 = kein Retry) |
| `delay` | `int` | `200` | Wartezeit in ms zwischen Retries (blockierend) |
| `runOnce` | `bool` | `false` | Bei Re-Runs einer bestehenden Instanz wird das gespeicherte Ergebnis wiederverwendet statt den Step erneut auszuführen — für Seiteneffekte, die nicht doppelt passieren dürfen (Zahlung, E-Mail, externe Buchung) |

`retries`, `delay` und `runOnce` fließen in den Schema-Hash ein — eine Änderung erfordert eine neue Flow-Version.

## FlowBuilder Validierungsregeln

Im Constructor bzw. bei `addStep()`:

1. **Type-String-Format**: muss `/^flow\..+\.v\d+$/` matchen
2. **Init/Return-Klassen**: implementieren `MessageInitInterface` bzw. `MessageReturnInterface`
3. **Keine Duplikate**: dieselbe Step-Klasse darf nicht zweimal per `addStep()` hinzugefügt werden

Bei `build()`:

4. **Init-Message konsumiert**: mindestens ein Step hat die Init-Message als Constructor-Parameter
5. **Return-Message produziert**: falls deklariert, steht sie in `returnTypes()` mindestens eines Steps
6. **Kein Zyklus** im Step-Graph (DFS)
7. **Alle Steps erreichbar** vom Init-Step aus (BFS über Message-Ketten)
8. **Keine hängenden DataMessages**: jede `MessageDataInterface` in irgendeinem `returnTypes()` wird von mindestens einem Step konsumiert — **ohne Ausnahme für Seitenzweige**

Nicht validiert (Konvention, manuell prüfen): ein Produzent pro Message-Typ, nur eine mögliche Return-Message.

Details und Fehlermeldungen: `../create-flow/references/flowbuilder-validation.md`

## Flow-Attribute

- **`#[FlowEphemeral(expiryDays: 14)]`** — Flow ohne Primary-Storage-Persistierung, nur SQLite-Service-Index. Für Health-Checks, Monitoring, temporäre Flows. Details: `references/framework-concepts.md`
- **`#[FlowGroup('name')]`** — Gruppiert Flows im Dashboard (beeinflusst den Schema-Hash nicht). Details: `references/framework-concepts.md`

Beide nur hinzufügen wenn der User es wünscht oder das Projekt sie bereits verwendet.

## EmptyInitMessage — Flow ohne externen Input

Braucht ein Flow keinen externen Input (z.B. rein scheduler-getriggert), **keine eigene Init-Message erstellen**, sondern `Wundii\Flowcrafter\EmptyInitMessage` verwenden. Im ersten Step **muss** der Parameter `public readonly` sein (nicht `private`), damit Rector den scheinbar unbenutzten Parameter nicht entfernt:

```php
use Wundii\Flowcrafter\EmptyInitMessage;

// Flow
$flowBuilder = new FlowBuilder('flow.my-flow.v1', EmptyInitMessage::class);

// Erster Step
class MyFirstStep implements StepInterface
{
    public function __construct(
        public readonly EmptyInitMessage $init,  // public readonly — Pflicht!
    ) {}
}
```

## Dependency Injection

Flowcrafter baut **immer seinen eigenen** Symfony-`ContainerBuilder` — auch in Symfony-Anwendungen wird der App-Container **nicht** verwendet. Jeder Service, den ein Step, Schedule oder Projection-Handler im Constructor erwartet, muss deshalb in der `DependencyRegistry` registriert sein (Produktion: `flowcrafter.php` via `$flowcrafterConfig->setDependencyRegistry(...)`; Tests: `dependencyRegistry`-Parameter). Interfaces brauchen ein `bind()`. Details: `references/framework-concepts.md`.

## Code-Defaults

Beim Generieren von Flowcrafter-Code immer:

- `declare(strict_types=1)` setzen
- Messages: `readonly class` mit Constructor Property Promotion, Properties **`public`** (kein Getter)
- Steps: reguläre Klasse, `private readonly` Constructor-Parameter (Ausnahme: `EmptyInitMessage` → `public readonly`)
- Flows: reguläre Klasse mit `public static function schema(): FlowSchema`
- Schedules: reguläre Klasse mit `public function process(): void`
- Projections: reguläre Klasse, Handler-Methoden `public function on{Name}(FlowMessageReadonly $flowMessage): void`
- Alle Parameter und Return-Types explizit typisieren
- Suffixe: `*Message`, `*Step`, `*Flow`, `*Schedule`, `*Projection`

## Projekt-Kontext erkennen

Vor der Code-Generierung in einem unbekannten Projekt:

1. `composer.json` lesen — `wundii/flowcrafter` bestätigen, PHP-Version und PSR-4-Autoload (Namespace ↔ Verzeichnis) ermitteln
2. Glob auf `src/**/*Flow.php`, `src/**/*Step.php`, `src/**/*Message.php` → bestehende Verzeichnis- und Namespace-Konvention übernehmen
3. Eine bestehende Datei des jeweiligen Typs lesen → Code-Stil (`final`, `readonly`, Imports) bestätigen
4. `flowcrafter.php` lesen → wie werden Services in der `DependencyRegistry` registriert?

Gibt es noch keine Flowcrafter-Klassen: `src/Flowcrafter/{Flows,Steps,Messages,Schedules,Projections}/` als Vorschlag nennen und den User bestätigen lassen.

## Discovery

Schedules und Projection-Handler werden automatisch gefunden (Composer-Classmap + PSR-4-Verzeichnisse außerhalb `vendor/`). Ergebnisse werden pro Prozess gecacht — neue Klassen werden erst nach Neustart von Scheduler/Worker erkannt (im `dev`-Modus automatisch per File-Watcher). Dateien in PSR-4-Verzeichnissen werden per `require_once` geladen und dürfen keine Seiteneffekte beim Laden haben.

## Weiterführende Details

- `references/execution-model.md` — Rekursive Ausführung, Branching, Convergence, Fehlerverhalten
- `references/testing.md` — FlowTestCase, runFlow(), runStep(), Assertions, Fehlerpfade, Queue, Projections
- `references/data-mapper.md` — `wundii/data-mapper` für typsicheres Mapping von JSON/Array/XML → DTOs
- `references/framework-concepts.md` — Lifecycle, Retry, runOnce, Ephemeral, FlowGroup, Versionierung & Hashing, DI, Imports
