# Flowcrafter — Detaillierte Framework-Konzepte

> **Source of Truth:** Falls `vendor/wundii/flowcrafter/docs/concepts.md` bzw. `configuration.md` existiert, bei Detailfragen dort nachlesen.

## Message Routing Mechanismus

Flowcrafter baut beim `addStep()`/`build()` eine `messageToStepsMap` aus den Constructor-Signaturen der Steps:

1. Für jeden Step reflektiert die Engine dessen Constructor (ohne Constructor → Exception)
2. Parameter deren (benannter) Typ `MessageInterface` implementiert → **Message-Dependencies** (auto-injected aus dem Flow-Message-Pool). Union-/Intersection-Types werden ignoriert.
3. Alle anderen Parameter → **Service-Dependencies** (per DI-Container aufgelöst)
4. `returnTypes()` wird auf einer Instanz **ohne Constructor-Aufruf** gelesen — muss deshalb ein konstantes Array liefern
5. Jeder Message-Typ wird auf die Steps gemappt, die ihn konsumieren → `messageToStepsMap` (in `addStep()`-Reihenfolge)
6. Mehrere Steps können denselben Message-Typ konsumieren (Branching)
7. Ein Step kann mehrere Message-Typen benötigen (Convergence) — er wird erst ausgeführt wenn alle verfügbar sind

**Wichtige Einschränkung**: Zwei Steps im selben Flow dürfen nicht denselben Message-Typ produzieren, da Messages typbasiert geroutet und injiziert werden. Der `FlowBuilder` validiert das nicht.

## Flow Execution Lifecycle

1. Aufrufer übergibt eine Init-Message an den `FlowRunner`
2. `FlowRunner` erstellt `Flow`-Instanz; bei bestehender Instanz (`flowHash`) wird der Schema-Hash geprüft — weicht er ab, ist der Flow nicht ausführbar
3. `executeStepsRecursive` wird mit der Init-Message aufgerufen
4. Für die Message werden alle konsumierenden Steps aus der `messageToStepsMap` ermittelt
5. Pro Step: `executableMessages`-Check — sind **alle** benötigten Messages verfügbar?
6. Wenn ja → Step instantiieren (DI + Message-Injection) → `process()` aufrufen (bzw. bei `runOnce` + Re-Run gespeichertes Ergebnis wiederverwenden)
7. Ergebnis-Handling:
   - `MessageDataInterface` → rekursiver Aufruf von `executeStepsRecursive` mit der neuen Message
   - `MessageReturnInterface` → als Flow-Return gespeichert (nur der erste zählt)
   - `bool` → als `FlowResult` gespeichert, Zweig endet
   - Exception (Retries erschöpft) → `FlowException` gespeichert, Flow persistiert, Exception weitergeworfen
8. Wenn alle Rekursionen abgeschlossen → Flow persistiert, Return-Message (oder `false`) an den Aufrufer

**Message-Status**: `WAIT` → `PROCESS` → `FINISH`

Echo-/print-Ausgaben eines Steps werden abgefangen und als `FlowOutput` am Run gespeichert.

## Sync vs. Async Ausführung

**FlowRunner** (synchron):
```php
$runner = new FlowRunner(
    type: $schema->type(),
    flowSource: WeatherComfortFlow::class,
    flowSubject: null,       // optional
    storage: $storage,       // optional
    queue: $queue,           // optional — nötig, wenn Steps enqueue() nutzen
    dependencyRegistry: (new DependencyRegistry())
        ->instance($service1)
        ->instance($service2),
);
$result = $runner->run(message: new CityRequestMessage('Tokyo'));
// run(message, ?flowHash, ?queueId, includeSteps)
```

**FlowObserver** (asynchron, `vendor/bin/flowcrafter observer`):
- Long-running Worker der die Queue pollt
- Deserialisiert die Init-Message per DataMapper und verarbeitet sie mit einem frischen `FlowRunner`
- Für produktive Workloads empfohlen

**Schedules** (`vendor/bin/flowcrafter scheduler`):
- `$this->enqueue()` → asynchron via Queue (empfohlen)
- `$this->run()` → synchron, blockierend (braucht Storage im Schedule-Kontext)

**Steps die Sub-Flows triggern** (`extends AbstractStep`):
- Dieselben Methoden wie bei Schedules: `$this->enqueue()` (async) und `$this->run()` (sync)
- Signatur: `enqueue(flowSource, message, ?flowHash = null, ?flowSubject = null, includeSteps = [])`, `run(flowSource, message, ?flowHash = null, ?flowSubject = null): bool|MessageReturnInterface`
- Storage, Queue, DependencyRegistry und Projection-Handler werden vom laufenden Runner übernommen
- `enqueue()` ohne Queue → `RuntimeException`; `run()` hat keinen Zyklenschutz über Flow-Grenzen

## Retry-Mechanismus

Steps die externen I/O durchführen (HTTP-Calls, DB-Zugriffe) können automatisch bei Fehlern wiederholt werden:

```php
$flowBuilder->addStep(FetchWeatherStep::class, retries: 3, delay: 500);
```

- `retries: 3` → bis zu 3 **zusätzliche** Versuche (insgesamt 4 Ausführungen)
- `delay: 500` → 500ms Pause zwischen den Versuchen (blockierend, `usleep`)
- Default: `retries: 0, delay: 200`

**Verhalten bei Retry:**
1. `process()` wirft eine Exception
2. Flowcrafter erzeugt einen `FlowRetry`-Eintrag (attempt, message, stepSource, time)
3. Wartet `delay` ms
4. Erstellt eine **neue** Step-Instanz und ruft `process()` erneut auf
5. Nach dem letzten Fehlversuch wird eine `FlowException` persistiert und die Exception weitergeworfen

**FlowRetry-Einträge** dokumentieren jeden fehlgeschlagenen Zwischenversuch. Sie sind über `$flow->getFlowRetries()` abrufbar.

## runOnce — Step-Ergebnis bei Re-Runs wiederverwenden

```php
$flowBuilder->addStep(ChargeCreditCardStep::class, runOnce: true);
```

Bei einem **Re-Run** einer bestehenden Instanz (Storage + `flowHash`) wird ein `runOnce`-Step nicht erneut ausgeführt, wenn es aus einem früheren Run bereits ein Ergebnis gibt — eine erzeugte Message eines seiner `returnTypes` oder ein `FlowResult` dieses Steps. Das gespeicherte Ergebnis wird weitergereicht. Typischer Einsatz: Seiteneffekte, die nicht doppelt passieren dürfen (Zahlung, E-Mail, externe Buchung). Ohne Storage oder beim ersten Run läuft der Step normal. `runOnce` fließt in den Schema-Hash ein.

## Ephemeral Flows

Flows mit `#[FlowEphemeral]` überspringen alle Primary-Storage-Schreibvorgänge (Instanz, Run, Messages, Results, Retries, Exceptions) — nur das Schema wird registriert.

```php
#[FlowEphemeral(expiryDays: 7)]   // Default: 14, Minimum: 1
class HealthCheckFlow implements FlowInterface { /* ... */ }
```

**Storage-Verhalten:**
- `appendFlow()` wird mit `ephemeral: true` und `ephemeralExpiryDays: N` aufgerufen
- Nur der SQLite Service-Index wird beschrieben: `flow_list`, `flow_run_list`, `flow_exception_list` + `flow_ephemeral_list` (JSON-Kopie des kompletten Flows)

**Cleanup:**
- `FlowScheduler::tick()` ruft `cleanupEphemeral()` auf — **läuft kein Scheduler, wird nicht aufgeräumt**
- Abgelaufene Einträge (`expires_at <= now`) werden aus allen Index-Tabellen gelöscht

**API-Verhalten:**
- Ephemeral-Flows sind über die API lesbar, aber `isReadOnly: true`, `isExecutable: false` (nicht erneut ausführbar)
- Gehen bei `storage:rebuild --clear` verloren
- Projektionen funktionieren wie bei persistenten Flows

**Testing:**
- `FlowRunner` ohne Storage läuft ephemeral Flows problemlos
- Mit `StorageInterface`-Mock: `appendFlow` wird einmal mit `ephemeral: true` aufgerufen, alle anderen write-Methoden `never()`

## FlowGroup — Flows im Dashboard gruppieren

Das optionale `#[FlowGroup]`-Attribut (`Wundii\Flowcrafter\Attribute\FlowGroup`) gruppiert Flows im Flowcrafter-Dashboard. Es beeinflusst den Schema-Hash nicht.

```php
use Wundii\Flowcrafter\Attribute\FlowGroup;

#[FlowGroup('weather')]
class WeatherComfortFlow implements FlowInterface
{
    // ...
}
```

Nur hinzufügen wenn der User explizit eine Gruppierung wünscht oder das Projekt bereits `#[FlowGroup]` verwendet.

## Korrekte Imports

```php
use Wundii\Flowcrafter\AbstractMessage;
use Wundii\Flowcrafter\AbstractStep;
use Wundii\Flowcrafter\EmptyInitMessage;
use Wundii\Flowcrafter\FlowBuilder;
use Wundii\Flowcrafter\FlowMessageReadonly;
use Wundii\Flowcrafter\FlowSchema;
use Wundii\Flowcrafter\Interface\FlowInterface;
use Wundii\Flowcrafter\Interface\MessageInitInterface;
use Wundii\Flowcrafter\Interface\MessageDataInterface;
use Wundii\Flowcrafter\Interface\MessageReturnInterface;
use Wundii\Flowcrafter\Interface\ProjectionHandlerInterface;
use Wundii\Flowcrafter\Interface\StepInterface;
use Wundii\Flowcrafter\Attribute\FlowEphemeral;
use Wundii\Flowcrafter\Attribute\FlowGroup;
use Wundii\Flowcrafter\Attribute\FlowProjection;
use Wundii\Flowcrafter\Attribute\FlowProjectionMessage;
use Wundii\Flowcrafter\Attribute\FlowSchedule;
use Wundii\Flowcrafter\Schedule\AbstractSchedule;
use Wundii\Flowcrafter\DependencyInjection\DependencyRegistry;
use Wundii\Flowcrafter\Env;
```

## Versionierung, Schema-Hash & Read-Only

**Schema-Hash:** MD5 über das serialisierte Schema — Type, alle Steps mit ihren konsumierten Messages, `returnTypes`, `retries`, `delay`, `runOnce` **und die Hashes der beteiligten Messages**. `#[FlowGroup]` zählt nicht.

**Message-Hash:** MD5 über die sortierten Namen der promoted Properties (inkl. Kurzname verschachtelter Klassentypen). Umbenennen/Hinzufügen/Entfernen einer Property ändert den Hash, eine reine Typänderung skalarer Properties nicht.

**Ausführbarkeit:** Eine gespeicherte Flow-Instanz ist nur ausführbar (Re-Run, Retry über API, `runOnce`, `includeSteps`), wenn ihr `flowSchemaHash` dem aktuellen Code entspricht. Fehlen Klassen oder passen Message-Hashes nicht mehr, wird der Flow **read-only** geladen (Messages als `ReadonlyMessage`, `readOnlyReasons` erklärt warum).

**Wann `v<N>` erhöhen:** Bei **jeder** Änderung, die den Schema-Hash verändert:

- Step hinzugefügt, entfernt oder ersetzt
- Konsumierte Messages oder `returnTypes()` eines Steps geändert
- `retries`, `delay` oder `runOnce` geändert
- Property einer beteiligten Message umbenannt, hinzugefügt oder entfernt
- Init- oder Return-Message geändert

Kein Bump nötig bei: reinen Logik-Änderungen in `process()`, `#[FlowGroup]`, Service-Änderungen. (Der Step-Quellcode wird zusätzlich als Snapshot gespeichert, beeinflusst die Ausführbarkeit aber nicht.)

Mehrere Versionen können gleichzeitig existieren (alte Instanzen bleiben lesbar). **Nach einem Bump** alle Stellen anpassen, die den Type-String hart referenzieren: `#[FlowProjection(...)]`-Handler, Tests (`runFlow(flowType: ...)`), externe API-Aufrufe.

## DI-Integration

Flowcrafter nutzt intern Symfony `ContainerBuilder` (`symfony/dependency-injection`) für Autowiring. Es gibt **kein Symfony-Bundle** — Flowcrafter ist eine eigenständige Library und baut **immer seinen eigenen Container**. Auch in einer Symfony-Anwendung werden Services des App-Containers **nicht** automatisch verfügbar; unregistrierte Constructor-Abhängigkeiten führen zu einem Container-Fehler.

Service-Dependencies für Steps, Schedules und Projection-Handler werden über die fluent `DependencyRegistry` konfiguriert:

| Methode | Verhalten |
|---|---|
| `instance(object)` | Synthetic: fertige Instanz, gebunden an die **eigene Klasse** (nicht an ihre Interfaces) |
| `bind(string $id, object\|class-string)` | Interface-Binding: Objekt → synthetic + Alias; class-string → Autowire + Alias |
| `autowire(class-string)` | Eine einzelne Klasse per Name autowiren |
| `autowireNamespace(string)` | Jede instanziierbare Klasse unter einem PSR-4-Namespace-Prefix autowiren |
| `autowireDirectory(string)` | Jede instanziierbare Klasse unter einem Verzeichnis autowiren |
| `factory(class-string, Closure, ?alias)` | Lazy: Closure erhält den PSR-11-Container und liefert den Service (optionaler Alias) |

```php
use Wundii\Flowcrafter\DependencyInjection\DependencyRegistry;

$registry = (new DependencyRegistry())
    ->bind(HttpClientInterface::class, new CurlHttpClient()) // Interface → Objekt (+ Alias)
    ->instance(new MyLogger())                               // synthetic, an eigene Klasse gebunden
    ->autowire(SomeService::class)                           // einzelne Klasse autowiren
    ->autowireNamespace('App\\Service');                     // ganzen Namespace autowiren
```

**Produktion:** in der `flowcrafter.php` registrieren:

```php
return static function (FlowcrafterConfig $flowcrafterConfig): void {
    // ...
    $flowcrafterConfig->setDependencyRegistry(
        (new DependencyRegistry())
            ->autowireNamespace('App\\Service'),
    );
};
```

**Tests:** über den `dependencyRegistry`-Parameter von `runFlow()`/`runStep()` (siehe `testing.md`).

Autowiring registriert nur **Definitionen** (lazy, shared) — Services werden erst beim ersten `get()` instanziiert. Rohe Scalar-Config (API-Keys, Hosts) am besten in kleine Value-Objekte verpacken und per `instance()` registrieren, damit der Graph voll autowirebar bleibt; `factory()` ist dann selten nötig.

**`Env`-Helper** (`Wundii\Flowcrafter\Env`) — typisierter Reader für Environment-Variablen statt `(string) getenv('KEY') ?: 'default'`: `Env::string($key, $default)`, `Env::int($key, $default)`, `Env::bool($key, $default)`. Jeder fällt auf den Default zurück, wenn die Variable nicht gesetzt **oder** leer ist.

Dieselbe `DependencyRegistry` wird durch `FlowRunner`, `FlowScheduler`, `FlowObserver`, `ProjectionWorker`, `AbstractSchedule`, `AbstractStep` (für Sub-Flows) und `FlowAssertTrait` (Tests) gefädelt; aufgelöst in `FlowContainerFactory::build()`.

## Testing

Siehe `testing.md` — `FlowTestCase`, `runFlow()`, `runStep()`, Assertions, Fehlerpfade, Queue- und Projection-Tests.
