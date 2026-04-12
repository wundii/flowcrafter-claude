---
name: create-flow
description: >
  Generate a Flowcrafter Flow class. Use when the user asks to
  "create a flow", "add a flow", "new flow", "define a workflow",
  "build a processing pipeline", or wants to implement a FlowInterface
  class using the FlowBuilder DSL for Flowcrafter.
argument-hint: <flow-name> [init-message-class] [return-message-class]
allowed-tools: Read, Glob, Grep, Write
---

# Create Flowcrafter Flow

Generiere eine vollständige, validierte `FlowInterface`-Implementierung mit dem `FlowBuilder`-DSL.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<flow-name>` (Pflicht), `[init-message-class]` (optional), `[return-message-class]` (optional). Falls Argumente fehlen, frage den User nach dem Zweck des Flows, der Init-Message und der Return-Message bevor du generierst.

## Schritt 1: Projekt-Kontext erkennen

1. `composer.json` lesen → `wundii/flowcrafter`-Version bestätigen, PHP-Version prüfen
2. `symfony/framework-bundle` in `require` suchen → Symfony-Projekt erkannt
3. Glob auf `src/**/*Flow.php` → bestehende Flows für Namespace und Directory-Konvention
4. Einen bestehenden Flow lesen um das genaue Muster zu bestätigen (Variable-Name `$flowBuilder`, Import-Pfade)

## Schritt 2: Klasse und Platzierung bestimmen

- **Klassenname**: PascalCase + `Flow`-Suffix (z.B. `order-processing` → `OrderProcessingFlow`)
- **Directory**: bestehendes Pattern, sonst `src/Flowcrafter/Flows/` (Symfony) oder `src/Flow/` (pure PHP)
- **Namespace**: aus PSR-4 Autoload in `composer.json` ableiten
- **Type-String**: `flow.<kebab-case-name>.v1` (z.B. `flow.order-processing.v1`)

## Schritt 3: Validierungsregeln prüfen (VOR Codegenerierung)

Vor dem Schreiben des Codes sicherstellen:

- **Type-String-Format**: muss `/^flow\..+\.v\d+$/` matchen
- **Init-Message**: als zweites Argument an `FlowBuilder`. Klasse existiert oder User informieren
- **Return-Message**: als drittes Argument. Klasse existiert oder User informieren (optional)
- **Stubs**: mindestens ein Stub muss die Init-Message konsumieren (als Constructor-Parameter)
- User warnen wenn referenzierte Stubs oder Messages noch nicht existieren

## Schritt 4: Flow-Klasse generieren

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Flows;

use App\Flowcrafter\Messages\{InitMessageClass};
use App\Flowcrafter\Messages\{ReturnMessageClass};
use App\Flowcrafter\Stubs\{StubClass};
use Wundii\Flowcrafter\FlowBuilder;
use Wundii\Flowcrafter\FlowSchema;
use Wundii\Flowcrafter\Interface\FlowInterface;

class {ClassName}Flow implements FlowInterface
{
    public static function schema(): FlowSchema
    {
        $flowBuilder = new FlowBuilder(
            'flow.{name}.v1',
            {InitMessageClass}::class,
            {ReturnMessageClass}::class,
        );

        $flowBuilder->addStub({StubClass}::class);
        // Weitere Stubs in Abhängigkeitsreihenfolge

        return $flowBuilder->build();
    }
}
```

**Hinweise**:
- Variable heißt `$flowBuilder` (wie in den realen Beispielen)
- `addStub()` nimmt den FQCN als `::class` Konstante
- Reihenfolge der `addStub()`-Aufrufe ist für Dokumentation wichtig; Engine nutzt den Message-Graph
- Optional: `#[Wundii\Flowcrafter\Attribute\FlowGroup('group-name')]` auf die Klasse setzen um den Flow im Dashboard zu gruppieren — nur wenn der User es wünscht oder das Projekt es bereits verwendet

## EmptyInitMessage — Flow ohne externen Input

Falls der Flow keinen externen Input benötigt (z.B. ein rein scheduler-getriggerter Flow), **keine eigene Init-Message erstellen**. Stattdessen die eingebaute `Wundii\Flowcrafter\EmptyInitMessage` verwenden:

```php
use Wundii\Flowcrafter\EmptyInitMessage;

$flowBuilder = new FlowBuilder(
    'flow.my-flow.v1',
    EmptyInitMessage::class,
    ReturnMessage::class, // optional
);
```

Im ersten Stub muss `EmptyInitMessage` als `public readonly` deklariert werden — **nicht `private`** — damit Rector den scheinbar unbenutzten Parameter nicht entfernt:

```php
class MyFirstStub implements StubInterface
{
    public function __construct(
        public readonly EmptyInitMessage $init,  // public readonly — Pflicht!
    ) {}
}
```

## Schritt 5: Output

1. Generierten Code mit Erklärung anzeigen
2. Fehlende Stubs und Messages auflisten, `/create-stub` und `/create-message` empfehlen
3. Datei schreiben mit dem Write-Tool

## Detaillierte Validierungsregeln

Siehe `references/flowbuilder-validation.md` für alle Fehlermeldungen und Fix-Vorschläge.
