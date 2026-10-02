---
name: create-test
description: >
  Generate PHPUnit tests for Flowcrafter (wundii/flowcrafter) Flows, Steps or
  Projection handlers using FlowTestCase. Use when the user asks to "test a flow",
  "write a flow test", "add tests for the step", "create a FlowTestCase",
  "test the projection", or wants test coverage for a *Flow.php, *Step.php or
  *Projection.php class.
argument-hint: <flow-step-or-projection-class> [--run]
allowed-tools: Read, Glob, Grep, Write, Edit, Bash(vendor/bin/phpunit:*), Bash(php:*)
---

# Create Flowcrafter Test

Generiere PHPUnit-Tests auf Basis von `Wundii\Flowcrafter\Testing\FlowTestCase`.

API-Referenz (Parameter, Assertions, Patterns): `../flowcrafter/references/testing.md`. Falls `vendor/wundii/flowcrafter/docs/testing.md` existiert, diese bevorzugen.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als Klassenname oder Dateipfad eines Flows, Steps oder Projection-Handlers. Mit `--run` die erzeugten Tests anschließend ausführen. Fehlt das Argument, fragen was getestet werden soll.

## Schritt 1: Kontext erkennen

1. Zielklasse lokalisieren und lesen
2. `phpunit.xml(.dist)` und `composer.json` (`autoload-dev`) lesen → Test-Verzeichnis und Namespace (z.B. `tests/` → `App\Tests\`)
3. Glob auf bestehende Tests (`tests/**/*Test.php`) → Stil übernehmen: `#[Test]`-Attribut oder `test*`-Methoden, `final`, eigene Basisklasse mit `FlowAssertTrait`?
4. Prüfen ob schon ein Test für die Klasse existiert → dann per Edit ergänzen statt neu schreiben

### Für einen Flow zusätzlich

- `schema()` lesen: Type-String, Init-/Return-Message, alle Steps mit `retries`/`runOnce`
- Jeden Step lesen: konsumierte Messages, Service-Parameter, `returnTypes()`, Verzweigungen in `process()`, geworfene Exceptions, `enqueue()`/`run()`-Aufrufe
- Message-Graph skizzieren: Hauptkette, Seitenzweige (bool), Conditional Branches (mehrere Return-Types)

## Schritt 2: Testfälle planen

Dem User die geplanten Testfälle kurz auflisten. Für einen **Flow**:

| Testfall | Assertions |
|---|---|
| Happy Path | `assertFlowOk()`, `assertStepExecuted()` für die Hauptkette, `assertFlowReturned()` + Inhalt prüfen |
| Jeder Conditional Branch | `assertStepExecuted()` / `assertStepNotExecuted()` des jeweiligen Zweigs |
| Seitenzweig erfolgreich / fehlgeschlagen | `assertFlowBoolResultFrom(Step::class, true\|false)`, bei `false` → `assertFlowStatus(StatusEnum::WARNING)` |
| Fehlerpfad (Step wirft) | `runFlow()` in `try/catch`, danach `assertFlowFailed()` + `assertFlowExceptionFrom()` |
| Retry | Fake, der erst wirft und dann erfolgreich ist → `assertFlowOk()`, ggf. `lastFlow()->getFlowRetries()` zählen |
| Sub-Flow (`enqueue()`) | `InMemoryQueue` übergeben, eingereihte Items prüfen |

Für einen **Step**: `runStep()` mit allen konsumierten Messages; pro Verzweigung in `process()` einen Test, Return-Typ und -Inhalt prüfen.

Für einen **Projection-Handler**: Methode direkt mit `FlowMessageReadonly::createFromArray([...])` aufrufen (Keys = Property-Namen der Message), Read Model prüfen, zweiter Aufruf mit derselben Message → Idempotenz.

## Schritt 3: Abhängigkeiten ersetzen

- Externe Services (HTTP, ntfy, DB) durch Fakes/Stubs ersetzen — bevorzugt kleine In-Test-Klassen oder bestehende Fakes im Projekt; PHPUnit-Mocks nur wenn im Projekt üblich
- Über `dependencyRegistry: (new DependencyRegistry())` übergeben:
  - Interface-Parameter → `->bind(Interface::class, $fake)` (`instance()` bindet nur an die eigene Klasse!)
  - Konkrete Klassen → `->instance($fake)` oder `->autowire(Klasse::class)`
- **Jeder** Service-Parameter aller ausgeführten Steps muss auflösbar sein, sonst scheitert der Container
- Für `AbstractStep` mit `enqueue()` → `queue: new InMemoryQueue()`

## Schritt 4: Test generieren

```php
<?php

declare(strict_types=1);

namespace App\Tests\Flowcrafter\Flows;

use App\Flowcrafter\Flows\{FlowClass};
use App\Flowcrafter\Messages\{InitMessage};
use App\Flowcrafter\Messages\{ReturnMessage};
use App\Flowcrafter\Steps\{StepClass};
use PHPUnit\Framework\Attributes\Test;
use RuntimeException;
use Wundii\Flowcrafter\DependencyInjection\DependencyRegistry;
use Wundii\Flowcrafter\Testing\FlowTestCase;

final class {FlowClass}Test extends FlowTestCase
{
    private const FLOW_TYPE = 'flow.{name}.v1';

    #[Test]
    public function happyPathReturnsResult(): void
    {
        $this->runFlow(
            flowType: self::FLOW_TYPE,
            flowSource: {FlowClass}::class,
            initMessage: new {InitMessage}(/* ... */),
            dependencyRegistry: (new DependencyRegistry())
                ->bind(SomeClientInterface::class, new FakeSomeClient(/* ... */)),
        );

        $this->assertFlowOk();
        $this->assertStepExecuted({StepClass}::class);

        $result = $this->assertFlowReturned({ReturnMessage}::class);
        self::assertSame(/* expected */, $result->someProperty);
    }

    #[Test]
    public function failingStepMarksFlowFailed(): void
    {
        try {
            $this->runFlow(
                flowType: self::FLOW_TYPE,
                flowSource: {FlowClass}::class,
                initMessage: new {InitMessage}(/* ... */),
                dependencyRegistry: (new DependencyRegistry())
                    ->bind(SomeClientInterface::class, new FailingSomeClient()),
            );
            self::fail('Expected exception was not thrown.');
        } catch (RuntimeException) {
            // runFlow() wirft die Step-Exception weiter
        }

        $this->assertFlowFailed();
        $this->assertFlowExceptionFrom({StepClass}::class);
    }
}
```

Regeln:
- Type-String als Konstante aus dem Flow übernehmen (nicht raten) — nach einem Versions-Bump muss er angepasst werden
- Steps mit `retries` und `delay` verlangsamen Fehlerpfad-Tests (blockierender `delay`) — dem User das nennen
- Keine Assertions auf interne Reihenfolgen paralleler Zweige
- Keine echten Netzwerk-/DB-Zugriffe

## Schritt 5: Ausführen (bei `--run` oder auf Nachfrage)

```bash
vendor/bin/phpunit --filter {FlowClass}Test
```

Schlägt ein Test fehl: Ursache analysieren — Fehler im Test (falscher Fake, fehlende Registrierung) selbst korrigieren; deutet der Fehler auf einen Bug im Produktivcode hin, **nicht** den Produktivcode ändern, sondern dem User den Befund melden. Läuft PHP nur im Container (Docker), den passenden Befehl aus der Projekt-Doku (`CLAUDE.md`, `README.md`, `Makefile`) verwenden oder den User fragen.

## Schritt 6: Output

1. Geplante Testfälle und generierten Code anzeigen
2. Datei schreiben (Write bzw. Edit bei bestehendem Test)
3. Neu angelegte Fakes nennen
4. Bei `--run`: Ergebnis der Ausführung berichten
