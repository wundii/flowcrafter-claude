---
name: create-message
description: >
  Generate a Flowcrafter Message class. Use when the user asks to
  "create a message", "add a message", "new init message", "new data message",
  "new return message", or wants to define a new MessageInitInterface,
  MessageDataInterface, or MessageReturnInterface class for Flowcrafter
  (wundii/flowcrafter).
argument-hint: <message-name> <init|data|return> [property:type ...]
allowed-tools: Read, Glob, Grep, Write
---

# Create Flowcrafter Message

Generiere eine typisierte, immutable Message-Klasse für die Flowcrafter Workflow-Engine.

Konzepte und Code-Defaults: `../flowcrafter/SKILL.md` (Abschnitte „Messages“ und „EmptyInitMessage“).

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<message-name>` (Pflicht), `<init|data|return>` (Pflicht — Message-Typ), gefolgt von optionalen Property-Deklarationen im Format `name:type`. Falls der Message-Typ fehlt, frage den User.

**Init-Message ohne Properties?** Dann keine Klasse erzeugen, sondern die eingebaute `Wundii\Flowcrafter\EmptyInitMessage` empfehlen (siehe `../flowcrafter/SKILL.md` → „EmptyInitMessage“; im ersten Step `public readonly`).

## Message-Typen

| Interface | Namespace | Zweck | Regel |
|---|---|---|---|
| `MessageInitInterface` | `Wundii\Flowcrafter\Interface\MessageInitInterface` | Flow-Eintrittspunkt | Genau eine pro Flow. Wird vom ersten Step konsumiert. |
| `MessageDataInterface` | `Wundii\Flowcrafter\Interface\MessageDataInterface` | Zwischendaten | **Muss** von mindestens einem Step konsumiert werden — auch auf Seitenzweigen. Genau ein produzierender Step pro Flow. |
| `MessageReturnInterface` | `Wundii\Flowcrafter\Interface\MessageReturnInterface` | Terminaler Flow-Output | Genau eine pro Flow, produziert von der Hauptkette. Wird nicht weiter konsumiert. |

Braucht ein Seitenzweig ein Ergebnis? Keine neue Message, sondern `bool` im Step.

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Message.php` — bestehende Messages für Namespace und Directory
2. Eine bestehende Message lesen und Namespace-Prefix sowie Stil (`final`, public vs. Getter) bestätigen
3. Namespace aus `composer.json` PSR-4 Autoload ableiten falls keine Messages gefunden
4. Prüfen ob eine Message mit gleichem Namen existiert

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

### Pflichtregeln (Serialisierung & Rehydrierung)

Messages werden als JSON gespeichert und beim Laden (Observer, Re-Run, `runOnce`, `includeSteps`) per DataMapper **über den Constructor** wiederhergestellt. Daraus folgt:

- `readonly class` ist Pflicht (`AbstractMessage` ist `abstract readonly`)
- **Alle** Daten als promoted Constructor-Properties — nicht-promoted Properties werden nicht serialisiert
- Keine Logik im Constructor, die beim Rehydrieren andere Werte erzeugt (z.B. `new DateTimeImmutable()` als Default)
- Erlaubte Typen: Skalare, `?`-nullable, Arrays, Enums, `DateTimeInterface` (wird beim Laden standardmäßig als `DateTime` erzeugt), DTOs (siehe unten), andere Messages
- Array-Properties mit Objekten per PHPDoc typisieren (`/** @param Item[] $items */`), damit der DataMapper die Elemente mappen kann
- Property-Umbenennungen ändern den Message-Hash und damit den Schema-Hash aller Flows, die die Message nutzen → Flow-Version erhöhen

### DTOs als Message-Properties

Ein DTO muss denselben Round-Trip überstehen:

- **Einfachster Weg:** `final readonly class` mit `public` promoted Properties — wird von `json_encode` direkt korrekt serialisiert.
- **Mit `private` Properties:** `JsonSerializable` implementieren, und die Keys **exakt wie die Constructor-Parameter** benennen — sonst kann das DTO beim Laden nicht wiederhergestellt werden.

```php
final readonly class DeviceState implements \JsonSerializable
{
    public function __construct(
        private string $name,
        private float $temperature,
    ) {}

    /** @return array<string, mixed> */
    public function jsonSerialize(): array
    {
        return [
            'name' => $this->name,               // Key = Constructor-Parametername
            'temperature' => $this->temperature,
        ];
    }
}
```

Bei vielen Feldern oder Mapping aus externen APIs: `wundii/data-mapper` (`../flowcrafter/references/data-mapper.md`).

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

`private` promoted Properties werden ebenfalls serialisiert (Reflection) — der Round-Trip funktioniert.

## Schritt 3: Dateiname und Platzierung

- Directory: bestehendes Pattern; gibt es keins, `src/Flowcrafter/Messages/` vorschlagen
- Klassenname: PascalCase + `Message`-Suffix (z.B. `city-request` → `CityRequestMessage`)
- Dateiname: `{ClassName}Message.php`

## Schritt 4: Output

1. Generierten Code anzeigen und Interface-Wahl begründen
2. Datei schreiben mit dem Write-Tool
3. Hinweisen welcher Step diese Message produzieren und welche Steps sie konsumieren sollten
4. Falls Init- oder Return-Message: `create-flow` empfehlen um den Flow zu definieren
5. Falls eine bestehende Message geändert wurde: betroffene Flows nennen (Grep auf den Klassennamen) und auf Versionserhöhung hinweisen
