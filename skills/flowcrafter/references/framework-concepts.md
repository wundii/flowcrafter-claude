# Flowcrafter — Detaillierte Framework-Konzepte

## Message Routing Mechanismus

Flowcrafter baut einen internen Dependency-Graph aus den Constructor-Signaturen der Steps:

1. Für jeden Step reflektiert die Engine dessen Constructor
2. Parameter deren Typen `AbstractMessage` extenden oder ein Message-Interface implementieren → **Message-Dependencies** (auto-injected aus dem Flow-Message-Store)
3. Alle anderen Parameter → **Service-Dependencies** (per DI-Container aufgelöst)
4. Die Engine ordnet die Step-Ausführung so, dass wenn Step B die Message von Step A benötigt, Step A zuerst läuft
5. Rückgabewerte werden im Message-Store unter ihrem Klassennamen abgelegt

**Wichtige Einschränkung**: Zwei Steps im selben Flow dürfen nicht denselben Message-Typ produzieren, da der zweite den ersten im Store überschreiben würde.

## Flow Execution Lifecycle

1. Aufrufer übergibt eine Init-Message an den `FlowRunner`
2. `FlowRunner` erstellt `Flow`-Instanz, validiert Schema-Hash
3. Steps werden in Dependency-Reihenfolge instantiiert (DI + Message-Injection)
4. `process()` jedes Steps wird aufgerufen. Return-Wert wird im Message-Store abgelegt
5. Wenn der Step mit `MessageReturnInterface`-Return fertig ist → Flow beendet
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

## Retry-Mechanismus

Steps die externen I/O durchführen (HTTP-Calls, DB-Zugriffe) können automatisch bei Fehlern wiederholt werden:

```php
$flowBuilder->addStep(FetchWeatherStep::class, retries: 3, delay: 500);
```

- `retries: 3` → bis zu 3 **zusätzliche** Versuche (insgesamt 4 Ausführungen)
- `delay: 500` → 500ms Pause zwischen den Versuchen
- Default: `retries: 0, delay: 200`

**Verhalten bei Retry:**
1. `process()` wirft eine Exception
2. Flowcrafter erzeugt einen `FlowRetry`-Eintrag (attempt, message, stepSource, time)
3. Wartet `delay` ms
4. Erstellt eine **neue** Step-Instanz und ruft `process()` erneut auf
5. Nach dem letzten Fehlversuch (`retries` erschöpft) wird die Exception geworfen und als `FlowException` persistiert

**FlowRetry-Einträge** dokumentieren jeden fehlgeschlagenen Zwischenversuch im Flow. Sie sind über `$flow->getFlowRetries()` abrufbar und werden in allen Storage-Backends persistiert.

**Schema-Hash**: `retries` und `delay` sind Teil des Schema-Hashs. Eine Änderung dieser Werte erzwingt eine neue Flow-Version.

## Ephemeral Flows

Flows mit `#[FlowEphemeral]` überspringen alle Primary-Storage-Schreibvorgänge (Schema, Instanz, Run, Messages, Results, Sources). Der `FlowRunner` erkennt das Attribut per Reflection und setzt interne Flags (`ephemeral = true`, `ephemeralExpiryDays`).

**Storage-Verhalten:**
- `appendFlow()` wird mit `ephemeral: true` und `ephemeralExpiryDays: N` aufgerufen
- Nur der SQLite Service-Index wird beschrieben: `flow_list`, `flow_run_list`, `flow_exception_list` (wie normal) + `flow_ephemeral_list` (JSON-Kopie des kompletten Flows)
- Die `flow_ephemeral_list`-Tabelle speichert `flow_hash`, `flow_runtime_hash`, `flow_json`, `expires_at` und `time`

**Cleanup:**
- `FlowScheduler::tick()` ruft `cleanupEphemeral()` auf
- Abgelaufene Einträge (wo `expires_at <= now`) werden aus `flow_ephemeral_list`, `flow_list`, `flow_run_list` und `flow_exception_list` gelöscht
- Nicht-abgelaufene Einträge bleiben erhalten

**API-Verhalten:**
- `GET /api/flow/flow-details` prüft zuerst `flow_ephemeral_list`, dann Primary Storage
- Ephemeral-Antworten erhalten `isReadOnly: true`, `isExecutable: false` und `readOnlyReasons`-Array mit Ephemeral-Hinweis
- Der Sync-Drift-Check (`StoragePreflight`) subtrahiert ephemeral Flows vom Service-Count

**Testing:**
- `FlowRunner` ohne Storage läuft ephemeral Flows problemlos (alle Storage-Calls sind optional)
- Mit `StorageInterface`-Mock: `appendFlow` wird einmal mit `ephemeral: true` aufgerufen, alle anderen write-Methoden `never()`

## FlowSchema und Hashing

Jeder Flow hat einen Content-Hash (MD5 des Schemas). Beim Ausführen prüft Flowcrafter ob der Schema-Hash der gespeicherten Instanz mit dem aktuellen Code übereinstimmt. Änderungen am Flow (neue Steps, andere Messages, retries, delay) erzwingen eine neue Version (`v2`, `v3`...).

## DI-Integration

Flowcrafter nutzt intern Symfony `ContainerBuilder` (`symfony/dependency-injection`) für Autowiring. Es gibt **kein Symfony-Bundle** — Flowcrafter ist eine eigenständige Library.

Service-Dependencies für Steps und Schedules werden über das `dependenciesInjection`-Array konfiguriert — drei Modi:

| Schlüssel | Wert | Verhalten |
|---|---|---|
| ohne Schlüssel | `class-string` | Klasse wird per Autowiring registriert |
| ohne Schlüssel | `object` | Konkrete Instanz, gebunden an eigene Klasse |
| Interface-Klassenname | `object` | Instanz wird an Interface und konkrete Klasse gebunden (Alias) |

```php
$flowRunner = new FlowRunner(
    type: 'flow.order.v1',
    flowSource: OrderFlow::class,
    storage: $storage,
    dependenciesInjection: [
        HttpClientInterface::class => new CurlHttpClient(),
        new MyLogger(),
        SomeService::class,
    ],
);
```

Dasselbe Array gilt für `FlowcrafterConfig::setDependenciesInjection()` (Service/Observer), `FlowScheduler` und `FlowAssertTrait` (Tests).

## Flow-Versionierung

Das `v<N>` Suffix im Type-String zeigt Breaking Changes an:

- **Version erhöhen** wenn: Init-Message-Struktur ändert, Steps entfernt werden, Return-Message ändert
- **Gleiche Version** bei: optionalen neuen Steps, rein internen Änderungen
- Mehrere Versionen können gleichzeitig laufen (für schrittweise Migration)

## Testing

`FlowTestCase` — Storage-loser Unit-Test:

```php
class WeatherFlowTest extends FlowTestCase
{
    public function testFlow(): void
    {
        $this->runFlow(
            flowType: 'flow.weather-comfort.v2',
            flowSource: WeatherComfortFlow::class,
            initMessage: new CityRequestMessage('Tokyo'),
        );

        $this->assertFlowOk();
        $this->assertStepExecuted(FetchWeatherStep::class);
        $this->assertFlowReturned(WeatherReportMessage::class);
    }
}
```

Verfügbare Assertions:

| Assertion | Beschreibung |
|---|---|
| `assertFlowOk()` | Status ist `OK` |
| `assertFlowFailed()` | Status ist `FAILED` |
| `assertFlowStatus(StatusEnum)` | Beliebiger Status |
| `assertFlowReturned(class)` | Return-Message prüfen (gibt gecastete Instanz zurück) |
| `assertStepExecuted(class)` | Step wurde ausgeführt |
| `assertStepNotExecuted(class)` | Step wurde nicht ausgeführt |
| `assertFlowHasMessage(class)` | Message-Typ im Flow vorhanden |
| `assertFlowMessageCount(int)` | Anzahl Flow-Messages |
| `assertFlowBoolResult(bool)` | Alle FlowResults haben erwarteten Wert |
| `assertFlowBoolResultFrom(class, bool)` | FlowResult eines bestimmten Steps |
| `assertFlowResultCount(int)` | Anzahl FlowResults |
| `assertNoFlowExceptions()` | Keine Exceptions im Flow |
| `assertFlowExceptionFrom(class, ?msg)` | Exception von bestimmtem Step (optional mit Message-Substring) |
| `assertFlowRunCount(int)` | Anzahl Runs |

DI im Test via `dependencies`-Parameter:
```php
$this->runFlow(
    flowType: 'flow.weather-comfort.v2',
    flowSource: WeatherComfortFlow::class,
    initMessage: new CityRequestMessage('Tokyo'),
    dependencies: [
        HttpClientInterface::class => new FakeHttpClient(),
    ],
);
```
