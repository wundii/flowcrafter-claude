# Stub Patterns

## Einfacher Transformation-Stub (keine externen Services)

```php
class ConvertWeatherStub implements StubInterface
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

## Stub mit Service-Dependency (Symfony Autowiring)

```php
class FetchWeatherStub implements StubInterface
{
    public function __construct(
        private readonly CityRequestMessage $cityRequestMessage,  // Message — auto-injected
        private readonly OpenWeatherMapClient $apiClient,          // Service — DI-injected
    ) {}
}
```

In pure PHP: `$apiClient` muss im `dependenciesInjection`-Array des `FlowRunner` enthalten sein.

## Mehrere Return-Types (Conditional Branching)

Beide Types müssen downstream von einem Stub konsumiert werden (oder `MessageReturnInterface` sein):

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

Ein Stub kann mehrere Messages konsumieren — er wird erst ausgeführt wenn alle upstream Messages vorliegen:

```php
class ResultProcessStub implements StubInterface
{
    public function __construct(
        private readonly PvOutputMessage $pvOutput,          // von PvAnlageStub
        private readonly BatteryStatusMessage $batteryStatus, // von BatteriespeicherStub
    ) {}

    public function returnTypes(): array
    {
        return [EnergyReportMessage::class];
    }
}
```

Beide `PvOutputMessage` und `BatteryStatusMessage` müssen von anderen Stubs im selben Flow produziert werden.

## Terminaler Stub (Flow-Output)

Der Stub gibt die Return-Message des Flows zurück:

```php
class SummaryReportStub implements StubInterface
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

Stubs können auch `bool` zurückgeben — wird als `FlowResult` aufgezeichnet:

```php
public function process(): bool
{
    $success = $this->sendNotification();
    return $success; // true → OK, false → WARNING im Flow-Status
}
```

`returnTypes()` kann in diesem Fall `[]` zurückgeben oder weggelassen werden (leeres Array).

## Stub für pure PHP (kein DI-Container)

```php
class ManualServiceStub implements StubInterface
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
