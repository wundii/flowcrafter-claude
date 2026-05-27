# Flow Analysis Checklist

Erweiterte Checkliste für die Produktionsreife-Prüfung von Flowcrafter Flows.

## Strukturelle Korrektheit (Pflicht)

- [ ] Type-String matcht `/^flow\..+\.v\d+$/`
- [ ] Init-Message implementiert `MessageInitInterface`
- [ ] Return-Message implementiert `MessageReturnInterface` (falls deklariert)
- [ ] Mindestens ein Step konsumiert die Init-Message
- [ ] Keine doppelten `addStep()`-Aufrufe
- [ ] Kein Zyklus im Message-Dependency-Graph
- [ ] Alle Steps vom Init-Step erreichbar
- [ ] Alle `MessageDataInterface`-Return-Types downstream konsumiert

## Typsicherheit

- [ ] Alle Step Constructor-Parameter sind vollständig typisiert (kein `mixed`)
- [ ] Alle `process()`-Methoden haben expliziten Return-Type
- [ ] `returnTypes()` stimmt mit `process()` Return-Type(s) überein
- [ ] Message Constructor-Properties sind alle typisiert

## Idiomatisches PHP

- [ ] Messages: `readonly class` (PHP 8.2+) oder `readonly` Properties (8.1+)
- [ ] Steps: `private readonly` Constructor-Parameter
- [ ] Constructor Property Promotion genutzt (Messages) oder Getter-Methoden konsistent
- [ ] Kein mutabler Zustand in Steps oder Messages
- [ ] `declare(strict_types=1)` in allen Dateien

## Architektur

- [ ] Jeder Step hat genau eine, klare Verantwortung (Name beschreibt eine Aktion)
- [ ] Kein Step fetched externe Daten UND transformiert sie (Single Responsibility)
- [ ] Services im Step sind Interface-typisiert (nicht konkrete Klassen) für Testbarkeit
- [ ] Message-Property-Namen sind domänen-semantisch (nicht technisch)
- [ ] Flow-Version (`v1`, `v2`) reflektiert tatsächliche Breaking Changes

## Retry-Konfiguration

- [ ] Steps mit externem I/O (HTTP, DB, APIs) haben `retries` konfiguriert
- [ ] `delay` ist sinnvoll gewählt (z.B. 500ms+ für Rate-Limits, 200ms für Netzwerk-Glitches)
- [ ] Reine Transformations-Steps haben KEINE `retries` (deterministische Logik braucht keinen Retry)
- [ ] `retries`-Werte sind nicht übertrieben hoch (>5 ist selten sinnvoll)

## Observability und Testbarkeit

- [ ] Steps mit externem I/O (HTTP, DB) nutzen Interface-typisierte Service-Parameter
- [ ] Messages tragen genug Kontext für Logging/Debugging
- [ ] Type-String ist projektübergreifend eindeutig (Grep bestätigt keine Duplikate)
- [ ] FlowTestCase-Tests existieren für den Flow

## Ephemeral-Konfiguration

- [ ] Falls `#[FlowEphemeral]` gesetzt: `expiryDays` ist sinnvoll gewählt (nicht zu kurz für Debugging, nicht zu lang für Disk-Verbrauch)
- [ ] Falls `#[FlowEphemeral]` gesetzt: Flow wird nicht von anderen Flows als Abhängigkeit referenziert (ephemeral Flows haben keine Primary-Storage-Daten für Lookups)
- [ ] Falls `#[FlowEphemeral]` gesetzt: Flow produziert keine kritischen Business-Daten die dauerhaft persistiert werden müssen
- [ ] Nicht-ephemeral: kein `#[FlowEphemeral]` auf Flows die Business-Logik-Ergebnisse liefern

## Häufige Anti-Patterns

| Anti-Pattern | Problem | Lösung |
|---|---|---|
| God Step | Ein Step macht alles (fetch + transform + send) | Aufteilen in separate Steps |
| Zu viele Service-Dependencies | Step injiziert 5+ Services | Fachlich aufteilen |
| Unklares Naming | `ProcessStep`, `HandleStep` | Konkrete Aktion benennen: `FetchWeatherStep` |
| Fehlende `readonly` | Mutabler Zustand möglich | `private readonly` oder `readonly class` |
| Zu generische Messages | `DataMessage` mit 20 Properties | Domänen-spezifische Messages pro Kontext |
| Hard-coded Credentials in Steps | API-Keys direkt im Code | Config/Service-Injection nutzen |
| Ephemeral auf Business-Flow | Wichtige Daten nicht dauerhaft persistiert | `#[FlowEphemeral]` entfernen |
| Nicht-ephemeral auf Monitoring | Monitoring-Pings füllen Primary Storage | `#[FlowEphemeral]` setzen |
