---
name: create-step
description: >
  Generate a Flowcrafter Step class. Use when the user asks to
  "create a step", "add a step", "new step", "implement a processing step",
  or wants to implement a StepInterface class for a Flowcrafter workflow.
argument-hint: <step-name> [consumed-message] [--returns <message-class>]
allowed-tools: Read, Glob, Grep, Write
---

# Create Flowcrafter Step

Generiere eine `StepInterface`-Implementierung mit korrekter Constructor-basierter Message-Injection und `returnTypes()`-Deklaration.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse das erste Token als Step-Name (PascalCase, `Step`-Suffix wird hinzugefügt). Falls unklar, folgende Fragen stellen:

- "Welche Messages konsumiert dieser Step?" (Constructor-Parameter vom Message-Typ)
- "Welche Service-Abhängigkeiten braucht er?" (nicht-Message Constructor-Parameter)
- "Welche Message gibt er zurück?" (Output-Message-Klasse)
- "Ist dieser der terminale Step?" (gibt er `MessageReturnInterface` zurück?)

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Step.php` — bestehende Steps für Namespace und Directory
2. Einen bestehenden Step lesen und bestätigen:
   - Namespace-Prefix (z.B. `App\Flowcrafter\Steps`)
   - Verwendet `process()` konkreten Return-Type oder `MessageDataInterface`?
   - Klassen-Stil: `final`, `readonly`, oder reguläre Klasse?
3. `composer.json` auf Symfony prüfen

## Schritt 2: Constructor-Design-Regeln

Flowcrafter verwendet Reflection zur Auto-Discovery:

- **Message-Parameter**: Constructor-Parameter deren Typen `AbstractMessage` extenden oder Message-Interfaces implementieren → werden vom Engine aus dem Message-Store injiziert
- **Service-Parameter**: alle anderen Constructor-Parameter → werden per DI-Container aufgelöst (Symfony Autowiring oder `dependenciesInjection`-Array)
- Alle Parameter: `private readonly` — **Ausnahme**: `EmptyInitMessage` muss `public readonly` sein (siehe unten)
- Reihenfolge: Messages zuerst (Konvention aus bestehenden Beispielen)

### Sonderfall: EmptyInitMessage

Wenn der Step als erster Step eines Flows ohne externen Input agiert, muss `EmptyInitMessage` als `public readonly` deklariert werden — **nicht `private`** — damit Rector den scheinbar unbenutzten Parameter nicht entfernt:

```php
use Wundii\Flowcrafter\EmptyInitMessage;

class MyFirstStep implements StepInterface
{
    public function __construct(
        public readonly EmptyInitMessage $init,  // public readonly — Pflicht!
    ) {}
}
```

## Schritt 3: returnTypes() Regeln

- Array von FQCNs aller möglichen Return-Types
- Jede Klasse muss `MessageDataInterface` oder `MessageReturnInterface` implementieren
- `process()` Return-Type muss mit diesem Array übereinstimmen
- Bei mehreren möglichen Types: Union-Type in `process()` (z.B. `MessageA|MessageDataInterface`)

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

**Bei terminalem Step** (gibt Return-Message des Flows zurück):
```php
use Wundii\Flowcrafter\Interface\MessageReturnInterface;

public function process(): MessageReturnInterface
{
    return new {ReturnMessage}(/* ... */);
}
```

**Bei mehreren möglichen Return-Types**:
```php
public function returnTypes(): array
{
    return [SuccessMessage::class, FailureMessage::class];
}

public function process(): MessageDataInterface
{
    if ($this->someCondition()) {
        return new SuccessMessage(/* ... */);
    }
    return new FailureMessage(/* ... */);
}
```

## Schritt 5: Output

1. Generierten Code mit Erklärung anzeigen
2. Referenzierte Messages prüfen (existieren sie bereits?)
3. Datei schreiben mit dem Write-Tool
4. Empfehlen zu welchem Flow dieser Step hinzugefügt werden sollte

## Patterns und Beispiele

Siehe `references/step-patterns.md` für Multi-Return-Types, Fan-in, Service-Integration.
