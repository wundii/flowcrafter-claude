# Flowcrafter — Testing

## FlowTestCase

`Wundii\Flowcrafter\Testing\FlowTestCase` — abstrakte PHPUnit-TestCase-Klasse für Flow- und Step-Tests. Stellt `runFlow()`, `runStep()` und spezialisierte Assertions bereit.

```php
use Wundii\Flowcrafter\Testing\FlowTestCase;

final class MyFlowTest extends FlowTestCase
{
    // ...
}
```

## runFlow() — Flow-Integrationstest

Führt einen kompletten Flow ohne Storage aus. Alle Steps werden in Reihenfolge ausgeführt, Branching und Convergence funktionieren wie in Produktion.

```php
$this->runFlow(
    flowType: 'flow.openweathermap.v1',
    flowSource: OpenWeatherMapFlow::class,
    initMessage: new OpenWeatherMapRequestMessage(51.5833, 8.1667),
    dependencies: [
        HttpClientInterface::class => $mockHttpClient,
        new MockNtfyService(),
    ],
);
```

**Parameter:**

| Parameter      | Typ                           | Beschreibung                                 |
|----------------|-------------------------------|----------------------------------------------|
| `flowType`     | `string`                      | Flow-Type-String (z.B. `flow.my-flow.v1`)    |
| `flowSource`   | `class-string<FlowInterface>` | Flow-Klasse                                  |
| `initMessage`  | `MessageInterface`            | Init-Message für den Flow                    |
| `flowSubject`  | `?string`                     | Optionaler Business-Key                      |
| `dependencies` | `array`                       | DI-Abhängigkeiten (Services, Mocks)          |
| `includeSteps` | `class-string[]`              | Nur diese Steps ausführen (für Partial-Runs) |

**Return:** `bool|MessageReturnInterface` — das Flow-Ergebnis.

## runStep() — Isolierter Step-Test

Führt einen einzelnen Step ohne Flow-Kontext aus. Nutzt den gleichen DI-Container wie FlowRunner.

```php
$result = $this->runStep(
    stepSource: FetchWeatherDataStep::class,
    messages: [
        new OpenWeatherMapRequestMessage(51.5833, 8.1667),
    ],
    dependencies: [
        new OpenWeatherMapService($credentials, $mockHttpClient),
    ],
);
```

**Parameter:**

| Parameter      | Typ                           | Beschreibung                     |
|----------------|-------------------------------|----------------------------------|
| `stepSource`   | `class-string<StepInterface>` | Step-Klasse                      |
| `messages`     | `MessageInterface[]`          | Messages die der Step konsumiert |
| `dependencies` | `array`                       | DI-Abhängigkeiten                |

**Return:** `bool|MessageInterface` — das Step-Ergebnis.

## Assertions

### Flow-Status

| Assertion                              | Beschreibung                                          |
|----------------------------------------|-------------------------------------------------------|
| `assertFlowOk()`                       | Flow-Status ist `OK`                                  |
| `assertFlowFailed()`                   | Flow-Status ist `FAILED`                              |
| `assertFlowStatus(StatusEnum)`         | Beliebiger Status                                     |

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
        dependencies: [/* mocks with alerts in response */],
    );

    $this->assertFlowOk();
    $this->assertStepExecuted(FetchDataStep::class);
    $this->assertStepExecuted(SendAlertStep::class);       // Seitenzweig
    $this->assertStepExecuted(ForwardDataStep::class);      // Hauptkette
    $this->assertFlowBoolResultFrom(SendAlertStep::class, true);  // Notification gesendet
    $result = $this->assertFlowReturned(WeatherResultMessage::class);
    // Assertions auf $result->currentWeather etc.
}

#[Test]
public function flowWithoutAlertsSkipsNotification(): void
{
    $this->runFlow(
        flowType: 'flow.weather.v1',
        flowSource: WeatherFlow::class,
        initMessage: new WeatherRequestMessage(51.5, 8.1),
        dependencies: [/* mocks WITHOUT alerts */],
    );

    $this->assertFlowOk();
    $this->assertFlowBoolResultFrom(SendAlertStep::class, false);  // Keine Notification
}
```

## Wann FlowTestCase vs. Unit-Test

| Situation                              | Empfehlung                               |
|----------------------------------------|------------------------------------------|
| Flow-Gesamtverhalten (Happy Path)      | `FlowTestCase::runFlow()`                |
| Branching/Convergence korrekt?         | `FlowTestCase::runFlow()`                |
| Einzelner Step mit komplexer Logik     | `FlowTestCase::runStep()` oder Unit-Test |
| Service-Methoden isoliert              | Standard PHPUnit TestCase                |

## Import

```php
use Wundii\Flowcrafter\Testing\FlowTestCase;
use Wundii\Flowcrafter\Enum\StatusEnum;
```
