---
name: create-message
description: >
  Generate a Flowcrafter Message class. Use when the user asks to
  "create a message", "add a message", "new init message", "new data message",
  "new return message", or wants to define a new MessageInitInterface,
  MessageDataInterface, or MessageReturnInterface class for Flowcrafter.
argument-hint: <message-name> <init|data|return> [property:type ...]
allowed-tools: Read, Glob, Grep, Write
---

# Create Flowcrafter Message

Generiere eine typisierte, immutable Message-Klasse für die Flowcrafter Workflow-Engine.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<message-name>` (Pflicht), `<init|data|return>` (Pflicht — Message-Typ), gefolgt von optionalen Property-Deklarationen im Format `name:type`. Falls der Message-Typ fehlt, frage den User.

## Message-Typen

| Interface | Namespace | Zweck | Regel |
|---|---|---|---|
| `MessageInitInterface` | `Wundii\Flowcrafter\Interface\MessageInitInterface` | Flow-Eintrittspunkt | Genau eine pro Flow. Wird vom ersten Step konsumiert. |
| `MessageDataInterface` | `Wundii\Flowcrafter\Interface\MessageDataInterface` | Zwischendaten | Muss von einem downstream Step konsumiert werden. |
| `MessageReturnInterface` | `Wundii\Flowcrafter\Interface\MessageReturnInterface` | Terminaler Flow-Output | Genau eine pro Flow. Muss nicht weiter konsumiert werden. |

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Message.php` — bestehende Messages für Namespace und Directory
2. Eine bestehende Message lesen und Namespace-Prefix sowie PHP-Version bestätigen
3. Namespace aus `composer.json` PSR-4 Autoload ableiten falls keine Messages gefunden

## Schritt 2: Message-Klasse generieren

### Standard-Template — `public` Properties (Default)

Properties sind **standardmäßig `public`** — kein Getter nötig. Nur auf `private` + Getter wechseln wenn der User es explizit verlangt.

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Messages;

use Wundii\Flowcrafter\AbstractMessage;
use Wundii\Flowcrafter\Interface\MessageInitInterface;

readonly class {ClassName}Message extends AbstractMessage implements MessageInitInterface
{
    public function __construct(
        public string $propertyOne,
        public int $propertyTwo,
    ) {}
}
```

Passe `MessageInitInterface` an den gewählten Typ an (`MessageDataInterface` / `MessageReturnInterface`).

### Nur auf Wunsch des Users: `private` + Getter

```php
readonly class {ClassName}Message extends AbstractMessage implements MessageInitInterface
{
    public function __construct(
        private string $propertyOne,
    ) {}

    public function getPropertyOne(): string
    {
        return $this->propertyOne;
    }
}
```

## Schritt 3: EmptyInitMessage — kein Input benötigt

Falls ein Flow keinen externen Input braucht, **keine eigene Init-Message erstellen**. Stattdessen die eingebaute `Wundii\Flowcrafter\EmptyInitMessage` verwenden.

Im Flow:
```php
use Wundii\Flowcrafter\EmptyInitMessage;

$flowBuilder = new FlowBuilder(
    'flow.my-flow.v1',
    EmptyInitMessage::class,
    // kein Return-Message nötig
);
```

Im ersten Step: das `EmptyInitMessage`-Property **muss `public readonly`** sein (nicht `private`), damit statische Analyse-Tools (z.B. Rector) den "unbenutzten" Parameter nicht entfernen:

```php
class MyFirstStep implements StepInterface
{
    public function __construct(
        public readonly EmptyInitMessage $init,  // public readonly — Pflicht!
    ) {}
}
```

## Schritt 4: Dateiname und Platzierung

- Directory: bestehendes Pattern, sonst `src/Flowcrafter/Messages/` (Symfony) oder `src/Message/` (pure PHP)
- Klassenname: PascalCase + `Message`-Suffix (z.B. `city-request` → `CityRequestMessage`)
- Dateiname: `{ClassName}Message.php`

## Schritt 5: Output

1. Generierten Code anzeigen und Interface-Wahl begründen
2. Datei schreiben mit dem Write-Tool
3. Hinweisen welche Steps diese Message konsumieren sollten
4. Falls Init- oder Return-Message: `/create-flow` empfehlen um den Flow zu definieren
