---
name: create-step
description: >
  Generate a Flowcrafter Step class. Use when the user asks to
  "create a step", "add a step", "new step", "implement a processing step",
  or wants to implement a StepInterface / AbstractStep class for a Flowcrafter
  (wundii/flowcrafter) workflow.
argument-hint: <step-name> [consumed-message] [--returns <message-class>]
allowed-tools: Read, Glob, Grep, Write, Edit
---

# Create Flowcrafter Step

Generiere eine `StepInterface`-Implementierung mit korrekter Constructor-basierter Message-Injection und `returnTypes()`-Deklaration.

Konzepte und Code-Defaults: `../flowcrafter/SKILL.md` (insbesondere „Steps“, „Step-Design-Leitfaden“, „Dependency Injection“).

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse das erste Token als Step-Name (PascalCase, `Step`-Suffix wird hinzugefügt). Falls unklar, folgende Fragen stellen:

- "Welche Messages konsumiert dieser Step?" (Constructor-Parameter vom Message-Typ)
- "Welche Service-Abhängigkeiten braucht er?" (nicht-Message Constructor-Parameter)
- "Was gibt er zurück?" — DataMessage (Hauptkette), Return-Message (Flow-Ende) oder `bool` (Seitenzweig/Seiteneffekt)
- "Soll der Step einen weiteren Flow anstoßen?" (dann `extends AbstractStep` für `$this->enqueue()`/`$this->run()`)
- "Hat er einen Seiteneffekt, der bei Re-Runs nicht wiederholt werden darf?" (dann im Flow `runOnce: true`)

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Step.php` — bestehende Steps für Namespace und Directory
2. Einen bestehenden Step lesen und bestätigen:
   - Namespace-Prefix (z.B. `App\Flowcrafter\Steps`)
   - `implements StepInterface` oder `extends AbstractStep`?
   - Verwendet `process()` konkreten Return-Type oder `MessageDataInterface`?
   - Klassen-Stil: `final`, `readonly`, oder reguläre Klasse?
3. Die konsumierten Messages lesen (Property-Namen und -Typen) und den Ziel-Flow lesen: **produziert schon ein anderer Step denselben Message-Typ?** Dann eine eigene Message-Klasse vorschlagen
4. `flowcrafter.php` lesen → wie werden Services registriert?

## Schritt 2: Constructor-Design-Regeln

Flowcrafter verwendet Reflection zur Auto-Discovery:

- **Ein Constructor ist Pflicht** — ohne Constructor schlägt `addStep()` fehl
- **Message-Parameter**: Parameter deren Typ `MessageInterface` implementiert → werden vom Engine aus dem Message-Store injiziert. Nur einfache benannte Typen — keine Union-/Nullable-Kombinationen mit anderen Typen
- **Service-Parameter**: alle anderen Parameter → werden vom **Flowcrafter-eigenen** DI-Container aufgelöst. Auch in Symfony-Projekten müssen sie in der `DependencyRegistry` registriert sein (Interfaces per `bind()`). Neue Services dem User zur Registrierung in `flowcrafter.php` nennen
- Alle Parameter: `private readonly` — **Ausnahme**: `EmptyInitMessage` muss `public readonly` sein (sonst entfernt Rector den scheinbar unbenutzten Parameter)
- Reihenfolge: Messages zuerst, dann Services

## Schritt 3: returnTypes() Regeln

- Array von FQCNs aller möglichen Message-Return-Types; bei `bool`-Steps `[]`
- **Konstantes Array** — kein Zugriff auf `$this`-Properties (Flowcrafter ruft `returnTypes()` auf einer Instanz ohne Constructor-Aufruf auf)
- Jede Klasse muss `MessageDataInterface` oder `MessageReturnInterface` implementieren
- `process()` darf nur Typen zurückgeben, die hier stehen — und alles was hier steht, muss im Flow konsumiert werden (DataMessage) bzw. das Flow-Ergebnis sein (Return-Message)
- Return-Type von `process()`: konkrete Klasse, Interface (`MessageDataInterface`) oder Union (`SuccessMessage|FailureMessage`)

## Schritt 4: Step-Klasse generieren

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Steps;

use App\Flowcrafter\Messages\{ConsumedMessage};
use App\Flowcrafter\Messages\{ReturnMessage};
use Wundii\Flowcrafter\Interface\MessageDataInterface;
use Wundii\Flowcrafter\Interface\StepInterface;
// use App\Service\{SomeService};

class {ClassName}Step implements StepInterface
{
    public function __construct(
        private readonly {ConsumedMessage} ${consumedMessageVar},
        // private readonly {SomeService} ${serviceVar},
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [{ReturnMessage}::class];
    }

    public function process(): MessageDataInterface
    {
        // TODO: Logik implementieren
        return new {ReturnMessage}(/* ... */);
    }
}
```

**Bei terminalem Step** (gibt die Return-Message des Flows zurück — nur ein Step pro Flow):
```php
use Wundii\Flowcrafter\Interface\MessageReturnInterface;

public function process(): MessageReturnInterface
{
    return new {ReturnMessage}(/* ... */);
}
```

**Bei Seiteneffekt-Step** (Seitenzweig, endet mit `bool`):
```php
/** @return class-string[] */
public function returnTypes(): array
{
    return [];
}

public function process(): bool
{
    try {
        $this->notifier->send(/* ... */);
    } catch (NotifierException) {
        return false; // Flow läuft weiter, Status WARNING
    }

    return true; // auch wenn nichts zu tun war
}
```

Ohne try/catch bricht eine Exception den **gesamten** Flow ab (`FAILED`). Nur fangen, wenn der Fehler den Flow tatsächlich nicht abbrechen soll.

Weitere Varianten (mehrere Return-Types, Fan-in, Retry): `references/step-patterns.md`.

## Schritt 4b: Sub-Flows triggern (`extends AbstractStep`)

Soll der Step aus `process()` heraus einen **weiteren Flow** starten, extends er `Wundii\Flowcrafter\AbstractStep` (implementiert selbst `StepInterface` — `returnTypes()`/`process()` bleiben Pflicht):

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Steps;

use App\Flowcrafter\Flows\{SubFlowClass};
use App\Flowcrafter\Messages\{ConsumedMessage};
use App\Flowcrafter\Messages\{SubFlowInitMessage};
use App\Flowcrafter\Messages\{ReturnMessage};
use Wundii\Flowcrafter\AbstractStep;
use Wundii\Flowcrafter\Interface\MessageDataInterface;

class {ClassName}Step extends AbstractStep
{
    public function __construct(
        private readonly {ConsumedMessage} ${consumedMessageVar},
    ) {}

    /** @return class-string[] */
    public function returnTypes(): array
    {
        return [{ReturnMessage}::class];
    }

    public function process(): MessageDataInterface
    {
        $this->enqueue(
            flowSource: {SubFlowClass}::class,
            message: new {SubFlowInitMessage}(/* ... */),
            // flowSubject: $this->getFlowSubject(),
        );

        return new {ReturnMessage}(/* ... */);
    }
}
```

- `enqueue()` (async, empfohlen) braucht eine Queue im Runner — sonst `RuntimeException`. In Tests `InMemoryQueue` an `runFlow(queue: ...)` übergeben
- `run()` (sync) liefert `bool|MessageReturnInterface` des Sub-Flows, hat aber keinen Zyklenschutz zwischen Flows
- Signatur: `enqueue(flowSource, message, ?flowHash, ?flowSubject, includeSteps)` / `run(flowSource, message, ?flowHash, ?flowSubject)`
- Reines `implements StepInterface` bleibt der Default

## Schritt 5: Output

1. Generierten Code mit Erklärung anzeigen
2. Referenzierte Messages prüfen (existieren sie bereits?) — fehlende mit `create-message` anlegen lassen
3. Datei schreiben mit dem Write-Tool
4. Den Ziel-Flow nennen; auf Wunsch den `addStep()`-Aufruf per Edit ergänzen (mit `retries` bei externem I/O, `runOnce` bei nicht wiederholbaren Seiteneffekten) und auf die nötige Versionserhöhung hinweisen (siehe `create-flow` → „Bestehenden Flow ändern“)
5. Neue Service-Abhängigkeiten nennen, die in der `DependencyRegistry` registriert werden müssen
