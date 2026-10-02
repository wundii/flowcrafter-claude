# Step Patterns

## Einfacher Transformation-Step (keine externen Services)

```php
class ConvertWeatherStep implements StepInterface
{
    public function __construct(
        private readonly RawWeatherMessage $rawWeather,
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [WeatherDataMessage::class];
    }

    public function process(): MessageDataInterface
    {
        return new WeatherDataMessage(
            celsius: round($this->rawWeather->tempKelvin - 273.15, 1),
            city: $this->rawWeather->city,
        );
    }
}
```

## Step mit Service-Dependency

```php
class FetchWeatherStep implements StepInterface
{
    public function __construct(
        private readonly CityRequestMessage $cityRequestMessage,  // Message — auto-injected
        private readonly OpenWeatherMapClient $apiClient,          // Service — DI-injected
    ) {}

    // returnTypes(), process() ...
}
```

Der Service wird vom **Flowcrafter-eigenen** Container aufgelöst — unabhängig davon, ob die Host-App Symfony nutzt. Er muss in der `DependencyRegistry` registriert sein:

```php
// flowcrafter.php
$flowcrafterConfig->setDependencyRegistry(
    (new DependencyRegistry())
        ->autowire(OpenWeatherMapClient::class),      // Klasse autowiren
        // ->instance(new OpenWeatherMapClient(...))  // oder fertige Instanz
        // ->bind(WeatherClientInterface::class, OpenWeatherMapClient::class) // Interface
);
```

## Side-Effect-Branch (Branching mit bool)

Für Steps die nur Seiteneffekte ausführen (Notifications, Cache-Writes, Logging) und keine neuen Daten in den Flow einbringen — `bool` zurückgeben statt eine Message-Klasse zu erstellen:

```
ComparedMessage → [StoreStep]  → ResultMessage   (Hauptkette, terminal)
                → [AlertStep]  → bool            (Side-Effect-Branch)
```

Beide Steps konsumieren dieselbe Message (Branching). Eine eigene DataMessage als Ende des Seitenzweigs ist **nicht** möglich (unkonsumiert → `build()` schlägt fehl), eine zweite Return-Message würde mit dem Flow-Ergebnis konkurrieren. Ebenso wenig darf der Seitenzweig denselben Message-Typ produzieren, den er konsumiert, oder den ein anderer Step produziert.

```php
class AlertStep implements StepInterface
{
    public function __construct(
        private readonly ComparedMessage $comparedMessage,
        private readonly NtfyClient $ntfy,
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [];
    }

    public function process(): bool
    {
        if (!$this->comparedMessage->hasAlerts) {
            return true; // nichts zu tun — Normalfall, kein WARNING
        }

        try {
            $this->ntfy->send('Alert: ...');
        } catch (NtfyException) {
            return false; // Flow läuft weiter, Status WARNING
        }

        return true;
    }
}
```

**Achtung**: `false` bedeutet WARNING im Flow-Status — nicht "kein Alert gesendet". Normale Zustände (kein Alert nötig, Bedingung nicht erfüllt) müssen `true` zurückgeben. Eine ungefangene Exception bricht dagegen den **ganzen** Flow ab (auch die Hauptkette) — nur sinnvoll, wenn der Seiteneffekt geschäftskritisch ist.

## Step mit Retry (fehleranfällige externe Operationen)

Steps die HTTP-Calls, DB-Zugriffe oder andere fehleranfällige I/O durchführen, sollten im Flow mit `retries` und `delay` registriert werden:

```php
// Im Flow:
$flowBuilder->addStep(FetchWeatherStep::class, retries: 3, delay: 500);
```

Der Step selbst bleibt unverändert — die Retry-Logik ist im `FlowRunner` implementiert:
- Bei Exception wird `process()` erneut aufgerufen (neue Step-Instanz)
- Jeder Fehlversuch wird als `FlowRetry`-Eintrag im Flow persistiert
- Nach Erschöpfung aller Retries wird eine `FlowException` persistiert und die Exception weitergeworfen
- Der `delay` blockiert den Worker-Prozess

Damit Retries greifen, muss der Step bei Fehlern **werfen** (nicht `false` zurückgeben).

**Wann `retries` setzen:**
- HTTP/API-Calls (Netzwerk-Timeouts, Rate-Limits)
- Datenbank-Operationen (Deadlocks, Connection-Drops)
- Externe Service-Aufrufe (temporäre Nichtverfügbarkeit)

**Wann NICHT:**
- Reine Transformations-Steps (deterministische Logik)
- Validierungs-Steps (Fehler ist gewollt, kein Retry sinnvoll)
- Nicht-idempotente Aufrufe, bei denen ein Timeout trotzdem ausgeführt worden sein kann (z.B. Zahlung ohne Idempotency-Key)

## Step mit nicht wiederholbarem Seiteneffekt (runOnce)

```php
// Im Flow:
$flowBuilder->addStep(ChargeCreditCardStep::class, runOnce: true);
```

Bei einem Re-Run einer bestehenden Flow-Instanz wird der Step nicht erneut ausgeführt, wenn bereits ein Ergebnis (Message oder `FlowResult`) existiert — das gespeicherte Ergebnis wird weitergereicht. Für Zahlung, E-Mail-Versand, externe Buchungen.

## Mehrere Return-Types (Conditional Branching)

Alle Types müssen downstream von einem Step konsumiert werden (oder die eine Return-Message des Flows sein):

```php
/** @return class-string[] */
public function returnTypes(): array
{
    return [SunnyDayMessage::class, RainyDayMessage::class];
}

public function process(): SunnyDayMessage|RainyDayMessage
{
    if ($this->weather->cloudCover < 30) {
        return new SunnyDayMessage($this->weather->city);
    }

    return new RainyDayMessage($this->weather->city);
}
```

Nur der Zweig der tatsächlich produzierten Message läuft weiter. Ein nachgelagerter Step, der **beide** Messages konsumiert, würde nie ausgeführt.

## Fan-in: Mehrere Messages konsumieren

Ein Step kann mehrere Messages konsumieren — er wird erst ausgeführt wenn alle upstream Messages vorliegen:

```php
class ResultProcessStep implements StepInterface
{
    public function __construct(
        private readonly PvOutputMessage $pvOutput,          // von PvAnlageStep
        private readonly BatteryStatusMessage $batteryStatus, // von BatteriespeicherStep
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [EnergyReportMessage::class];
    }

    // process() ...
}
```

Beide `PvOutputMessage` und `BatteryStatusMessage` müssen von (verschiedenen) anderen Steps im selben Flow produziert werden.

## Terminaler Step (Flow-Output)

Der Step gibt die Return-Message des Flows zurück — pro Flow sollte genau ein Step das tun:

```php
class SummaryReportStep implements StepInterface
{
    public function __construct(
        private readonly ActivityPlanMessage $activityPlan,
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [WeatherReportMessage::class]; // implements MessageReturnInterface
    }

    public function process(): MessageReturnInterface
    {
        return new WeatherReportMessage(
            summary: $this->activityPlan->summary,
        );
    }
}
```

## Bool-Rückgabe (FlowResult)

Steps können `bool` zurückgeben — wird als `FlowResult` aufgezeichnet:

```php
public function process(): bool
{
    return $this->sendNotification(); // true → OK, false → WARNING im Flow-Status
}
```

`returnTypes()` muss trotzdem implementiert werden (Interface-Pflicht) und liefert in diesem Fall `[]`.

## Sub-Flow triggern (`extends AbstractStep`)

Siehe `../SKILL.md` → „Schritt 4b“. Kurzform:

```php
class DispatchOrderStep extends AbstractStep
{
    public function __construct(
        private readonly OrderValidatedMessage $order,
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [OrderDispatchedMessage::class];
    }

    public function process(): MessageDataInterface
    {
        $this->enqueue(
            flowSource: ShipmentFlow::class,
            message: new ShipmentRequestMessage($this->order->orderId),
            flowSubject: $this->order->orderId,
        );

        return new OrderDispatchedMessage(/* ... */);
    }
}
```

Storage, Queue und `DependencyRegistry` werden vom laufenden Runner übernommen. Ohne Queue wirft `enqueue()` eine `RuntimeException`.
