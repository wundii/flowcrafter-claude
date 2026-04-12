# Flow Analysis Checklist

Erweiterte Checkliste für die Produktionsreife-Prüfung von Flowcrafter Flows.

## Strukturelle Korrektheit (Pflicht)

- [ ] Type-String matcht `/^flow\..+\.v\d+$/`
- [ ] Init-Message implementiert `MessageInitInterface`
- [ ] Return-Message implementiert `MessageReturnInterface` (falls deklariert)
- [ ] Mindestens ein Stub konsumiert die Init-Message
- [ ] Keine doppelten `addStub()`-Aufrufe
- [ ] Kein Zyklus im Message-Dependency-Graph
- [ ] Alle Stubs vom Init-Stub erreichbar
- [ ] Alle `MessageDataInterface`-Return-Types downstream konsumiert

## Typsicherheit

- [ ] Alle Stub Constructor-Parameter sind vollständig typisiert (kein `mixed`)
- [ ] Alle `process()`-Methoden haben expliziten Return-Type
- [ ] `returnTypes()` stimmt mit `process()` Return-Type(s) überein
- [ ] Message Constructor-Properties sind alle typisiert

## Idiomatisches PHP

- [ ] Messages: `readonly class` (PHP 8.2+) oder `readonly` Properties (8.1+)
- [ ] Stubs: `private readonly` Constructor-Parameter
- [ ] Constructor Property Promotion genutzt (Messages) oder Getter-Methoden konsistent
- [ ] Kein mutabler Zustand in Stubs oder Messages
- [ ] `declare(strict_types=1)` in allen Dateien

## Architektur

- [ ] Jeder Stub hat genau eine, klare Verantwortung (Name beschreibt eine Aktion)
- [ ] Kein Stub fetched externe Daten UND transformiert sie (Single Responsibility)
- [ ] Services im Stub sind Interface-typisiert (nicht konkrete Klassen) für Testbarkeit
- [ ] Message-Property-Namen sind domänen-semantisch (nicht technisch)
- [ ] Flow-Version (`v1`, `v2`) reflektiert tatsächliche Breaking Changes

## Observability und Testbarkeit

- [ ] Stubs mit externem I/O (HTTP, DB) nutzen Interface-typisierte Service-Parameter
- [ ] Messages tragen genug Kontext für Logging/Debugging
- [ ] Type-String ist projektübergreifend eindeutig (Grep bestätigt keine Duplikate)
- [ ] FlowTestCase-Tests existieren für den Flow

## Häufige Anti-Patterns

| Anti-Pattern | Problem | Lösung |
|---|---|---|
| God Stub | Ein Stub macht alles (fetch + transform + send) | Aufteilen in separate Stubs |
| Zu viele Service-Dependencies | Stub injiziert 5+ Services | Fachlich aufteilen |
| Unklares Naming | `ProcessStub`, `HandleStub` | Konkrete Aktion benennen: `FetchWeatherStub` |
| Fehlende `readonly` | Mutabler Zustand möglich | `private readonly` oder `readonly class` |
| Zu generische Messages | `DataMessage` mit 20 Properties | Domänen-spezifische Messages pro Kontext |
| Hard-coded Credentials in Stubs | API-Keys direkt im Code | Config/Service-Injection nutzen |
