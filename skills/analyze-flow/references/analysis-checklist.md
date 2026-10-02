# Flow Analysis Checklist

Erweiterte Checkliste für die Produktionsreife-Prüfung von Flowcrafter Flows.

## Strukturelle Korrektheit (Pflicht — vom FlowBuilder validiert)

- [ ] Type-String matcht `/^flow\..+\.v\d+$/`
- [ ] Init-Message implementiert `MessageInitInterface`
- [ ] Return-Message implementiert `MessageReturnInterface` (falls deklariert) und steht in `returnTypes()` eines Steps
- [ ] Mindestens ein Step konsumiert die Init-Message
- [ ] Keine doppelten `addStep()`-Aufrufe
- [ ] Kein Zyklus im Step-Graph
- [ ] Alle Steps vom Init-Step erreichbar
- [ ] Alle `MessageDataInterface`-Return-Types werden konsumiert — auch auf Seitenzweigen

## Strukturelle Korrektheit (Pflicht — NICHT vom FlowBuilder validiert)

- [ ] Jeder Message-Typ hat genau einen Produzenten
- [ ] Nur ein Step liefert eine Return-Message (sonst „erste gewinnt“)
- [ ] `returnTypes()` deckt alle möglichen Rückgaben von `process()` ab
- [ ] `returnTypes()` ist ein konstantes Array (kein `$this`-Zugriff)
- [ ] Jeder Step hat einen Constructor
- [ ] Type-String ist projektweit eindeutig

## Typsicherheit

- [ ] Alle Step Constructor-Parameter sind vollständig typisiert (kein `mixed`, keine Union-Types bei Messages — die werden nicht als Message erkannt)
- [ ] Alle `process()`-Methoden haben expliziten Return-Type
- [ ] Message Constructor-Properties sind alle typisiert

## Messages & Serialisierung

- [ ] Messages sind `readonly class` (Pflicht — `AbstractMessage` ist `abstract readonly`)
- [ ] Alle Message-Daten liegen in **promoted** Constructor-Properties (nicht-promoted werden nicht serialisiert)
- [ ] DTOs in Messages überstehen den Round-Trip: `public` promoted Properties oder `JsonSerializable` mit Keys = Constructor-Parameternamen
- [ ] `DateTimeInterface`-Properties funktionieren mit `DateTime` (Default-Mapping beim Laden)

## Idiomatisches PHP

- [ ] Steps: `private readonly` Constructor-Parameter (Ausnahme `EmptyInitMessage`: `public readonly`)
- [ ] Constructor Property Promotion in Messages; Getter-Stil projektweit konsistent
- [ ] Kein mutabler Zustand in Steps oder Messages
- [ ] `declare(strict_types=1)` in allen Dateien

## Architektur

- [ ] Jeder Step hat genau eine, klare Verantwortung (Name beschreibt eine Aktion)
- [ ] Kein Step fetched externe Daten UND transformiert sie (Single Responsibility)
- [ ] Services im Step sind Interface-typisiert (nicht konkrete Klassen) für Testbarkeit
- [ ] Alle Service-Parameter sind in der `DependencyRegistry` (`flowcrafter.php`) registriert bzw. registrierbar — der Container der Host-App wird nicht verwendet
- [ ] Message-Property-Namen sind domänen-semantisch (nicht technisch)
- [ ] Logik verlässt sich nicht auf die Ausführungsreihenfolge paralleler Zweige

## Fehlerverhalten

- [ ] Seiteneffekt-Steps (Notification, Logging, Cache) fangen ihre Exceptions und geben `false` zurück, wenn der Flow weiterlaufen soll
- [ ] `false` wird nur bei echtem Fehlschlag zurückgegeben („nichts zu tun“ = `true`)
- [ ] Steps mit nicht wiederholbaren Seiteneffekten (Zahlung, Mail, externe Buchung) haben `runOnce: true`
- [ ] Steps mit `enqueue()` laufen nur in Runnern mit Queue; `run()`-Sub-Flows erzeugen keine Rekursion zwischen Flows

## Retry-Konfiguration

- [ ] Steps mit externem I/O (HTTP, DB, APIs) haben `retries` konfiguriert
- [ ] `delay` ist sinnvoll gewählt (z.B. 500ms+ für Rate-Limits, 200ms für Netzwerk-Glitches) — der Delay blockiert den Worker
- [ ] Reine Transformations- und Validierungs-Steps haben KEINE `retries`
- [ ] `retries`-Werte sind nicht übertrieben hoch (>5 ist selten sinnvoll)

## Versionierung & Projections

- [ ] Flow-Version (`v1`, `v2`) wurde bei jeder Schema-Änderung erhöht (Steps, Messages, `retries`/`delay`/`runOnce`)
- [ ] `#[FlowProjection]`-Handler abonnieren die aktuelle Version des Type-Strings
- [ ] Tests verwenden den aktuellen Type-String

## Observability und Testbarkeit

- [ ] Messages tragen genug Kontext für Logging/Debugging
- [ ] FlowTestCase-Tests existieren für den Flow (Happy Path, Fehlerpfad, Seitenzweige)

## Ephemeral-Konfiguration

- [ ] Falls `#[FlowEphemeral]` gesetzt: `expiryDays` ist sinnvoll gewählt (nicht zu kurz für Debugging, nicht zu lang für Disk-Verbrauch)
- [ ] Falls `#[FlowEphemeral]` gesetzt: Flow muss nicht erneut ausführbar sein (ephemeral Flows sind read-only)
- [ ] Falls `#[FlowEphemeral]` gesetzt: Flow produziert keine kritischen Business-Daten
- [ ] Falls `#[FlowEphemeral]` gesetzt: ein Scheduler läuft (sonst kein Cleanup)

## Häufige Anti-Patterns

| Anti-Pattern | Problem | Lösung |
|---|---|---|
| God Step | Ein Step macht alles (fetch + transform + send) | Aufteilen in separate Steps |
| Zu viele Service-Dependencies | Step injiziert 5+ Services | Fachlich aufteilen |
| Unklares Naming | `ProcessStep`, `HandleStep` | Konkrete Aktion benennen: `FetchWeatherStep` |
| Terminale DataMessage im Seitenzweig | `build()` schlägt fehl | `bool` zurückgeben |
| Zweite Return-Message im Seitenzweig | Kann das Flow-Ergebnis überschreiben | `bool` zurückgeben |
| Ungefangene Exception im Seitenzweig | Ganzer Flow `FAILED` | Fangen, `false` zurückgeben |
| Zwei Produzenten desselben Message-Typs | Nicht-deterministisches Routing | Eigene Message-Klasse pro Produzent |
| `returnTypes()` mit `$this`-Zugriff | Leere Return-Types bei Schema-Erzeugung | Konstantes Array |
| Zu generische Messages | `DataMessage` mit 20 Properties | Domänen-spezifische Messages pro Kontext |
| Hard-coded Credentials in Steps | API-Keys direkt im Code | Value-Objekt per `DependencyRegistry::instance()`, Werte via `Env` |
| Zahlung/Mail ohne `runOnce` | Doppelte Ausführung bei Re-Runs | `runOnce: true` |
| Ephemeral auf Business-Flow | Wichtige Daten nicht dauerhaft persistiert | `#[FlowEphemeral]` entfernen |
| Nicht-ephemeral auf Monitoring | Monitoring-Pings füllen Primary Storage | `#[FlowEphemeral]` setzen |
