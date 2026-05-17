# Step Patterns

## Einfacher Transformation-Step (keine externen Services)

```php
class ConvertWeatherStep implements StepInterface
{
    public function __construct(
        private readonly RawWeatherMessage $rawWeather,
    ) {}

    public function returnTypes(): array
    {
        return [WeatherDataMessage::class];
    }

    public function process(): MessageDataInterface
    {
        return new WeatherDataMessage(
            celsius: round(($this->rawWeather->getTempKelvin() - 273.15), 1),
            city: $this->rawWeather->getCity(),
        );
    }
}
```

## Step mit Service-Dependency (Symfony Autowiring)

```php
class FetchWeatherStep implements StepInterface
{
    public function __construct(
        private readonly CityRequestMessage $cityRequestMessage,  // Message — auto-injected
        private readonly OpenWeatherMapClient $apiClient,          // Service — DI-injected
    ) {}
}
```

In pure PHP: `$apiClient` muss im `dependenciesInjection`-Array des `FlowRunner` enthalten sein.

## Step mit Retry (fehleranfällige externe Operationen)

Steps die HTTP-Calls, DB-Zugriffe oder andere fehleranfällige I/O durchführen, sollten im Flow mit `retries` und `delay` registriert werden:

```php
// Im Flow:
$flowBuilder->addStep(FetchWeatherStep::class, retries: 3, delay: 500);
```

Der Step selbst bleibt unverändert — die Retry-Logik ist im `FlowRunner` implementiert:
- Bei Exception wird `process()` erneut aufgerufen (neue Step-Instanz)
- Jeder Fehlversuch wird als `FlowRetry`-Eintrag im Flow persistiert
- Nach Erschöpfung aller Retries wird die Exception als `FlowException` geworfen

**Wann `retries` setzen:**
- HTTP/API-Calls (Netzwerk-Timeouts, Rate-Limits)
- Datenbank-Operationen (Deadlocks, Connection-Drops)
- Externe Service-Aufrufe (temporäre Nichtverfügbarkeit)

**Wann NICHT:**
- Reine Transformations-Steps (deterministische Logik)
- Validierungs-Steps (Fehler ist gewollt, kein Retry sinnvoll)

## Mehrere Return-Types (Conditional Branching)

Beide Types müssen downstream von einem Step konsumiert werden (oder `MessageReturnInterface` sein):

```php
public function returnTypes(): array
{
    return [SunnyDayMessage::class, RainyDayMessage::class];
}

public function process(): MessageDataInterface
{
    if ($this->weather->getCloudCover() < 30) {
        return new SunnyDayMessage($this->weather->getCity());
    }
    return new RainyDayMessage($this->weather->getCity());
}
```

## Fan-in: Mehrere Messages konsumieren

Ein Step kann mehrere Messages konsumieren — er wird erst ausgeführt wenn alle upstream Messages vorliegen:

```php
class ResultProcessStep implements StepInterface
{
    public function __construct(
        private readonly PvOutputMessage $pvOutput,          // von PvAnlageStep
        private readonly BatteryStatusMessage $batteryStatus, // von BatteriespeicherStep
    ) {}

    public function returnTypes(): array
    {
        return [EnergyReportMessage::class];
    }
}
```

Beide `PvOutputMessage` und `BatteryStatusMessage` müssen von anderen Steps im selben Flow produziert werden.

## Terminaler Step (Flow-Output)

Der Step gibt die Return-Message des Flows zurück:

```php
class SummaryReportStep implements StepInterface
{
    public function __construct(
        private readonly ActivityPlanMessage $activityPlan,
    ) {}

    public function returnTypes(): array
    {
        return [WeatherReportMessage::class]; // implements MessageReturnInterface
    }

    public function process(): MessageReturnInterface
    {
        return new WeatherReportMessage(
            summary: $this->activityPlan->getSummary(),
        );
    }
}
```

## Bool-Rückgabe (FlowResult)

Steps können auch `bool` zurückgeben — wird als `FlowResult` aufgezeichnet:

```php
public function process(): bool
{
    $success = $this->sendNotification();
    return $success; // true → OK, false → WARNING im Flow-Status
}
```

`returnTypes()` kann in diesem Fall `[]` zurückgeben oder weggelassen werden (leeres Array).

## Step für pure PHP (kein DI-Container)

```php
class ManualServiceStep implements StepInterface
{
    public function __construct(
        private readonly SomeMessage $message,
        private readonly MyService $service, // muss in dependenciesInjection des FlowRunner sein
    ) {}
}

// FlowRunner instantiieren:
$runner = new FlowRunner(
    type: $schema->type(),
    flowSource: MyFlow::class,
    storage: $storage,
    dependenciesInjection: [new MyService()], // Service-Instanzen hier
);
```
