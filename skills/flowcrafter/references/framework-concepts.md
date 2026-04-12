# Flowcrafter — Detaillierte Framework-Konzepte

## Message Routing Mechanismus

Flowcrafter baut einen internen Dependency-Graph aus den Constructor-Signaturen der Stubs:

1. Für jeden Stub reflektiert die Engine dessen Constructor
2. Parameter deren Typen `AbstractMessage` extenden oder ein Message-Interface implementieren → **Message-Dependencies** (auto-injected aus dem Flow-Message-Store)
3. Alle anderen Parameter → **Service-Dependencies** (per DI-Container aufgelöst)
4. Die Engine ordnet die Stub-Ausführung so, dass wenn Stub B die Message von Stub A benötigt, Stub A zuerst läuft
5. Rückgabewerte werden im Message-Store unter ihrem Klassennamen abgelegt

**Wichtige Einschränkung**: Zwei Stubs im selben Flow dürfen nicht denselben Message-Typ produzieren, da der zweite den ersten im Store überschreiben würde.

## Flow Execution Lifecycle

1. Aufrufer übergibt eine Init-Message an den `FlowRunner`
2. `FlowRunner` erstellt `Flow`-Instanz, validiert Schema-Hash
3. Stubs werden in Dependency-Reihenfolge instantiiert (DI + Message-Injection)
4. `process()` jedes Stubs wird aufgerufen. Return-Wert wird im Message-Store abgelegt
5. Wenn der Stub mit `MessageReturnInterface`-Return fertig ist → Flow beendet
6. Die Return-Message wird an den Aufrufer zurückgegeben

**Message-Status**: `WAIT` → `PROCESS` → `FINISH`

## Sync vs. Async Ausführung

**FlowRunner** (synchron):
```php
$runner = new FlowRunner(
    type: $schema->type(),
    flowSource: WeatherComfortFlow::class,
    storage: $storage,
    dependenciesInjection: [$service1, $service2],
);
$result = $runner->run(message: new CityRequestMessage('Tokyo'));
```

**FlowObserver** (asynchron):
- Long-running Worker der `storage->observeQueue()` pollt
- Verarbeitet Queue-Items via frische `FlowRunner::run()` Instanzen
- Für produktive Workloads empfohlen

**Schedule Dispatcher**:
- `$this->enqueue()` → asynchron via Queue (empfohlen)
- `$this->run()` → synchron, blockierend

## FlowSchema und Hashing

Jeder Flow hat einen Content-Hash (MD5 des Schemas). Beim Ausführen prüft Flowcrafter ob der Schema-Hash der gespeicherten Instanz mit dem aktuellen Code übereinstimmt. Änderungen am Flow (neue Stubs, andere Messages) erzwingen eine neue Version (`v2`, `v3`...).

## Symfony-Integration

Flowcrafter nutzt intern Symfony-Komponenten (`symfony/dependency-injection`, `symfony/config`). In einem Symfony-Projekt:

- Stubs werden als Services registriert (via `autoconfigure: true` in `services.yaml`)
- Flowcrafter Symfony Bundle (falls vorhanden) registriert Flows und Schedules automatisch
- Service-Dependencies in Stub-Constructors werden per Symfony-Autowiring aufgelöst
- Schedule-Runner integriert sich mit Symfony Messenger oder Scheduler

Prüfe `config/bundles.php` auf Flowcrafter-Bundle-Einträge.

## Pure PHP (ohne Symfony)

Ohne Symfony muss die Anwendung:

1. Einen PSR-11 Container oder manuelle Service-Wiring bereitstellen
2. `dependenciesInjection: [...]` Array beim `FlowRunner` befüllen
3. `FlowScheduler::tick()` oder `FlowScheduler::run()` über einen Cron-Job aufrufen
4. Storage-Backend (MySQL/Redis/ESDB) konfigurieren

Prüfe `bootstrap.php`, `container.php` oder ähnliche Entry-Points.

## Flow-Versionierung

Das `v<N>` Suffix im Type-String zeigt Breaking Changes an:

- **Version erhöhen** wenn: Init-Message-Struktur ändert, Stubs entfernt werden, Return-Message ändert
- **Gleiche Version** bei: optionalen neuen Stubs, rein internen Änderungen
- Mehrere Versionen können gleichzeitig laufen (für schrittweise Migration)

## Testing

`FlowTestCase` — Storage-loser Unit-Test:

```php
class WeatherFlowTest extends FlowTestCase
{
    public function testFlow(): void
    {
        $result = $this->runFlow(
            WeatherComfortFlow::class,
            WeatherComfortFlow::class,
            new CityRequestMessage('Tokyo'),
        );

        $this->assertFlowOk();
        $this->assertStubExecuted(FetchWeatherStub::class);
        $this->assertFlowReturned(WeatherReportMessage::class);
    }
}
```

Verfügbare Assertions: `assertFlowOk()`, `assertStubExecuted()`, `assertFlowHasMessage()`, `assertFlowReturned()`, `assertFlowBoolResult()`.
