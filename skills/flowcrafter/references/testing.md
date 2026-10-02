# Flowcrafter — Testing

> **Source of Truth:** Falls `vendor/wundii/flowcrafter/docs/testing.md` existiert, diese Datei bevorzugt lesen — sie passt immer zur installierten Version.

## FlowTestCase

`Wundii\Flowcrafter\Testing\FlowTestCase` — abstrakte PHPUnit-TestCase-Klasse für Flow- und Step-Tests. Stellt `runFlow()`, `runStep()` und spezialisierte Assertions bereit. Die Logik steckt in `Wundii\Flowcrafter\Testing\FlowAssertTrait` — wer bereits eine eigene Basisklasse hat, bindet stattdessen den Trait ein.

```php
use Wundii\Flowcrafter\Testing\FlowTestCase;

final class MyFlowTest extends FlowTestCase
{
    // ...
}
```

## runFlow() — Flow-Integrationstest

Führt einen kompletten Flow aus (standardmäßig ohne Storage und ohne Queue). Branching und Convergence funktionieren wie in Produktion.

```php
$this->runFlow(
    flowType: 'flow.openweathermap.v1',
    flowSource: OpenWeatherMapFlow::class,
    initMessage: new OpenWeatherMapRequestMessage(51.5833, 8.1667),
    dependencyRegistry: (new DependencyRegistry())
        ->bind(HttpClientInterface::class, $mockHttpClient)
        ->instance(new MockNtfyService()),
);
```

**Parameter:**

| Parameter            | Typ                           | Default                    | Beschreibung                                              |
|----------------------|-------------------------------|----------------------------|-----------------------------------------------------------|
| `flowType`           | `string`                      | —                          | Flow-Type-String (z.B. `flow.my-flow.v1`)                 |
| `flowSource`         | `class-string<FlowInterface>` | —                          | Flow-Klasse                                               |
| `initMessage`        | `MessageInterface`            | —                          | Init-Message für den Flow                                 |
| `flowSubject`        | `?string`                     | `null`                     | Optionaler Business-Key                                   |
| `storage`            | `?StorageInterface`           | `null`                     | Nur für Storage-Integrationstests                         |
| `queue`              | `?QueueInterface`             | `null`                     | **Pflicht**, sobald ein Step `$this->enqueue()` aufruft    |
| `dependencyRegistry` | `DependencyRegistry`          | `new DependencyRegistry()` | DI-Registry (Services, Mocks) — fluent                    |
| `includeSteps`       | `class-string[]`              | `[]`                       | Nur diese Steps (+ alle nachgelagerten) ausführen         |

**Return:** `bool|MessageReturnInterface` — das Flow-Ergebnis.

**Wichtig — Exceptions werden weitergeworfen:** Wirft ein Step (nach Erschöpfung aller Retries), wirft auch `runFlow()`. Der Flow ist dank `try/finally` trotzdem über `lastFlow()` und die Assertions verfügbar — siehe [Fehlerpfade testen](#fehlerpfade-testen).

## runStep() — Isolierter Step-Test

Führt einen einzelnen Step ohne Flow-Kontext aus. Nutzt dieselbe Container-Logik (`FlowContainerFactory`) wie der FlowRunner.

```php
$result = $this->runStep(
    stepSource: FetchWeatherDataStep::class,
    messages: [
        new OpenWeatherMapRequestMessage(51.5833, 8.1667),
    ],
    dependencyRegistry: (new DependencyRegistry())
        ->instance(new OpenWeatherMapService($credentials, $mockHttpClient)),
);
```

**Parameter:**

| Parameter                | Typ                           | Beschreibung                                                     |
|--------------------------|-------------------------------|------------------------------------------------------------------|
| `stepSource`             | `class-string<StepInterface>` | Step-Klasse                                                      |
| `messages`               | `MessageInterface[]`          | **Alle** Messages, die der Step im Constructor konsumiert        |
| `dependencyRegistry`     | `DependencyRegistry`          | DI-Registry — fluent                                             |
| `flowHash`, `flowRuntimeHash`, `flowType`, `flowSchemaHash`, `flowSubject` | `?string` | Nur für `AbstractStep` — Defaults werden generiert |
| `storage`, `queue`       | `?StorageInterface`, `?QueueInterface` | Nur für `AbstractStep`; `queue` nötig für `enqueue()` |
| `projectionHandlerMetas` | `ProjectionHandlerMeta[]`     | Nur für `AbstractStep` mit Sub-Flows + Projections               |

**Return:** `bool|MessageInterface` — das Step-Ergebnis. Es findet **keine** Rekursion statt, nachgelagerte Steps laufen nicht.

## Assertions

Alle Assertions akzeptieren optional als letzten Parameter `?Flow $flow` — ohne Angabe wird der Flow des letzten `runFlow()` verwendet.

### Flow-Status

| Assertion                              | Beschreibung                                          |
|----------------------------------------|-------------------------------------------------------|
| `assertFlowOk()`                       | Flow-Status ist `OK`                                  |
| `assertFlowFailed()`                   | Flow-Status ist `FAILED`                              |
| `assertFlowStatus(StatusEnum)`         | Beliebiger Status (z.B. `StatusEnum::WARNING` wenn ein bool-Step `false` lieferte) |

### Flow-Ergebnis

| Assertion                               | Beschreibung                                            |
|-----------------------------------------|---------------------------------------------------------|
| `assertFlowReturned(class)`             | Return-Message ist Instanz der Klasse (gibt sie zurück) |
| `assertFlowBoolResult(bool)`            | Alle FlowResults haben den erwarteten Wert              |
| `assertFlowBoolResultFrom(class, bool)` | FlowResult eines bestimmten Steps                       |
| `assertFlowResultCount(int)`            | Anzahl FlowResults (bool-Returns)                       |

### Steps und Messages

| Assertion                              | Beschreibung                                          |
|----------------------------------------|-------------------------------------------------------|
| `assertStepExecuted(class)`            | Step wurde ausgeführt                                 |
| `assertStepNotExecuted(class)`         | Step wurde NICHT ausgeführt                           |
| `assertFlowHasMessage(class)`          | Message-Typ im Flow vorhanden                         |
| `assertFlowMessageCount(int)`          | Anzahl Flow-Messages                                  |

### Exceptions

| Assertion                              | Beschreibung                                           |
|----------------------------------------|--------------------------------------------------------|
| `assertNoFlowExceptions()`             | Keine Exceptions im Flow                               |
| `assertFlowExceptionFrom(class, ?msg)` | Exception von bestimmtem Step (optional mit Substring) |

### Runs

| Assertion                              | Beschreibung                                          |
|----------------------------------------|-------------------------------------------------------|
| `assertFlowRunCount(int)`              | Anzahl Runs                                           |

## Helper-Methoden

| Methode        | Beschreibung                                                |
|----------------|-------------------------------------------------------------|
| `lastFlow()`   | Gibt den Flow der letzten `runFlow()`-Ausführung zurück     |
| `lastResult()` | Gibt das Ergebnis der letzten `runFlow()`-Ausführung zurück |

## Fehlerpfade testen

`runFlow()` wirft die Step-Exception weiter. Daher **immer** abfangen und danach auf dem Flow assertieren — sonst bricht der Test vor `assertFlowFailed()` ab:

```php
#[Test]
public function validationFailsOnEmptySku(): void
{
    try {
        $this->runFlow(
            flowType: 'flow.order.v1',
            flowSource: OrderFlow::class,
            initMessage: new OrderInitMessage(''),
        );
        self::fail('Expected RuntimeException was not thrown.');
    } catch (RuntimeException $exception) {
        self::assertStringContainsString('sku required', $exception->getMessage());
    }

    $this->assertFlowFailed();
    $this->assertFlowExceptionFrom(ValidateOrderStep::class, 'sku required');
}
```

Alternativ `$this->expectException(...)` — dann sind aber keine Assertions auf dem Flow mehr möglich.

## Steps mit enqueue() / run() testen

Steps die `AbstractStep` erweitern und `$this->enqueue()` aufrufen, brauchen eine Queue — ohne Queue wirft `enqueue()` eine `RuntimeException('Queue is not set.')`. Im Test die mitgelieferte `InMemoryQueue` verwenden:

```php
use Wundii\Flowcrafter\Queue\InMemoryQueue;

$queue = new InMemoryQueue();

$this->runFlow(
    flowType: 'flow.order.v1',
    flowSource: OrderFlow::class,
    initMessage: new OrderInitMessage('sku-42'),
    queue: $queue,
);

$items = iterator_to_array($queue->findAllQueues(), false);
$this->assertCount(1, $items);
$this->assertSame(ShipmentFlow::class, $items[0]->getFlowSource());
```

`$this->run()` (synchroner Sub-Flow) funktioniert auch ohne Storage und Queue — der Sub-Flow läuft dann ebenfalls in-memory.

## Projection-Handler testen

Handler sind normale Klassen — direkt mit einer `FlowMessageReadonly` aufrufen:

```php
use Wundii\Flowcrafter\FlowMessageReadonly;

$flowMessage = FlowMessageReadonly::createFromArray([
    'flowHash' => 'flow-hash', 'flowRuntimeHash' => 'runtime-hash', 'flowType' => 'flow.order.v1',
    'stepSource' => ValidateOrderStep::class, 'stepHash' => 'abc',
    'messageType' => 'finish', 'messageSource' => OrderValidatedMessage::class,
    'messageHash' => 'def', 'message' => ['sku' => 'sku-42', 'quantity' => 1],
    'time' => '2026-10-01T12:00:00.000+00:00', 'hash' => 'message-hash', 'predecessorHash' => null,
]);

(new OrderProjection($repository))->onValidated($flowMessage);
```

Idempotenz testen: dieselbe `FlowMessageReadonly` zweimal übergeben und prüfen, dass das Read Model unverändert bleibt.

## Pattern: Flow mit Branching testen

Wenn ein Flow Seitenzweige mit `bool`-Return hat (z.B. Benachrichtigungs-Steps), können sowohl der Hauptpfad als auch die Seitenzweige getestet werden:

```php
#[Test]
public function flowWithAlertsSendsNotification(): void
{
    $this->runFlow(
        flowType: 'flow.weather.v1',
        flowSource: WeatherFlow::class,
        initMessage: new WeatherRequestMessage(51.5, 8.1),
        dependencyRegistry: (new DependencyRegistry())
            ->instance(/* mock with alerts in response */),
    );

    $this->assertFlowOk();
    $this->assertStepExecuted(FetchDataStep::class);
    $this->assertStepExecuted(SendAlertStep::class);       // Seitenzweig
    $this->assertStepExecuted(ForwardDataStep::class);      // Hauptkette
    $this->assertFlowBoolResultFrom(SendAlertStep::class, true);
    $result = $this->assertFlowReturned(WeatherResultMessage::class);
    // Assertions auf $result->currentWeather etc.
}

#[Test]
public function failingNotificationSetsWarning(): void
{
    $this->runFlow(
        flowType: 'flow.weather.v1',
        flowSource: WeatherFlow::class,
        initMessage: new WeatherRequestMessage(51.5, 8.1),
        dependencyRegistry: (new DependencyRegistry())
            ->instance(/* ntfy mock that fails */),
    );

    $this->assertFlowStatus(StatusEnum::WARNING);
    $this->assertFlowBoolResultFrom(SendAlertStep::class, false);
}
```

`false` bedeutet „Seiteneffekt fehlgeschlagen“ → Status `WARNING`. „Kein Alert nötig“ ist ein Normalfall und liefert `true`.

## Teilweise Ausführung mit includeSteps

```php
$this->runFlow(
    flowType: 'flow.order.v1',
    flowSource: OrderFlow::class,
    initMessage: new OrderInitMessage('sku-42'),
    includeSteps: [ValidateOrderStep::class],
);
```

Die Liste wird automatisch um alle **nachgelagerten** Steps erweitert. Ohne Storage gibt es keine historischen Messages — Fan-in-Steps, deren zweiter Input außerhalb der Auswahl entsteht, laufen im Test daher nicht.

## Wann FlowTestCase vs. Unit-Test

| Situation                              | Empfehlung                               |
|----------------------------------------|------------------------------------------|
| Flow-Gesamtverhalten (Happy Path)      | `FlowTestCase::runFlow()`                |
| Branching/Convergence korrekt?         | `FlowTestCase::runFlow()`                |
| Fehlerpfad / Retry                     | `runFlow()` in `try/catch` + `assertFlowFailed()` |
| Einzelner Step mit komplexer Logik     | `FlowTestCase::runStep()` oder Unit-Test |
| Projection-Handler                     | Direkter Aufruf mit `FlowMessageReadonly` |
| Service-Methoden isoliert              | Standard PHPUnit TestCase                |

## Import

```php
use Wundii\Flowcrafter\Testing\FlowTestCase;
use Wundii\Flowcrafter\Enum\StatusEnum;
use Wundii\Flowcrafter\DependencyInjection\DependencyRegistry;
use Wundii\Flowcrafter\Queue\InMemoryQueue;
use Wundii\Flowcrafter\FlowMessageReadonly;
```
