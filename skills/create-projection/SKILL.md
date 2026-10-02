---
name: create-projection
description: >
  Generate a Flowcrafter Projection handler class. Use when the user asks to
  "create a projection", "add a projection", "new projection handler",
  "build a read model", "project flow messages", "add FlowProjection",
  "react to flow messages async", or wants to implement a
  ProjectionHandlerInterface class that consumes a Flowcrafter (wundii/flowcrafter)
  Flow's messages asynchronously.
argument-hint: <projection-name> <flow-type-or-class> [message-class ...]
allowed-tools: Read, Glob, Grep, Write, Edit
---

# Create Flowcrafter Projection

Generiere einen Projection-Handler, der die Messages eines Flows **asynchron** und entkoppelt verarbeitet — typischerweise um Read Models aufzubauen, Benachrichtigungen zu versenden oder Side-Effects auszulösen.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<projection-name>` (PascalCase, `Projection`-Suffix wird hinzugefügt, Pflicht), `<flow-type-or-class>` (Flow-Type-String wie `flow.order.v1` **oder** der Flow-Klassenname, Pflicht), danach beliebig viele `<message-class>` (Message-Sources, auf die je eine Handler-Methode gebunden wird). Fehlt etwas, frage nach:

- "Welchen Flow-Typ projiziert dieser Handler?" (Type-String `flow.<name>.v<N>` — pro Flow-Typ ist genau **ein** Handler zulässig)
- "Auf welche Messages soll reagiert werden?" (eine Methode pro Message-Source)
- "Was macht der Handler mit jeder Message?" (Read Model schreiben, Notification, Side-Effect)
- "Welche Service-Abhängigkeiten braucht er?" (Constructor-Parameter — z.B. ein Repository, HTTP-Client)

## Schritt 1: Projekt-Kontext erkennen

1. Glob auf `src/**/*Projection.php` — bestehende Projections für Namespace und Directory
2. Falls vorhanden, eine bestehende Projection lesen und das Pattern bestätigen (Namespace-Prefix, Klassen-Stil, Methoden-Naming wie `onValidated`)
3. Den referenzierten Flow lokalisieren (Glob auf `*{FlowClass}.php`) und ermitteln:
   - Den exakten Flow-Type-String aus `FlowBuilder('flow.<name>.v<N>', ...)`
   - Die Message-Klassen (Init/Data/Return), die als Sources in Frage kommen, und deren Property-Namen (= Keys in `getRawData()`)
4. **Prüfen ob bereits ein Handler den Flow-Typ abonniert** (Grep `#[FlowProjection` + Type-String). Falls ja: die neue Methode dort per Edit ergänzen statt eine zweite Klasse anzulegen — zwei Handler für denselben Flow-Typ werfen bei der Discovery

## Schritt 2: FlowProjection Attribut (Klasse)

```php
#[FlowProjection('flow.order.v1')]                     // einzelner Flow-Typ
#[FlowProjection(['flow.order.v1', 'flow.order.v2'])]  // mehrere Flow-Typen (z.B. über Versionen hinweg)
```

| Regel | Bedeutung |
|---|---|
| Mindestens ein Flow-Typ | leeres Array wirft bei der Discovery |
| Ein Handler pro Flow-Typ | derselbe Flow-Typ darf **nicht** von zwei Handlern abonniert werden |
| Type-String, nicht Klasse | das Attribut erwartet den Type-String (`flow.order.v1`), nicht den FQCN des Flows |
| Exakte Version | `flow.order.v1` empfängt **keine** Messages von `flow.order.v2` — bei Versions-Bumps beide Typen listen, solange alte Instanzen noch laufen |

## Schritt 3: FlowProjectionMessage Attribut (Methode)

Jede Handler-Methode bindet sich per `#[FlowProjectionMessage(MessageSource::class)]` an einen Message-Source. Regeln (werden bei der Discovery validiert):

- Die Methode muss `public` sein
- Sie muss **einen Parameter vom Typ `FlowMessageReadonly`** deklarieren
- Pro Handler darf ein Message-Source nur **einmal** gebunden werden (Duplikat wirft)
- Das Attribut ist wiederholbar — eine Methode kann auf mehrere Message-Sources reagieren
- Return-Type ist `void` (der Rückgabewert wird nicht ausgewertet)

### Zugriff auf die Message-Daten

Der Handler bekommt eine `FlowMessageReadonly` — die Original-Message-Klasse wird **nicht** instanziiert. Payload und Metadaten:

```php
public function onValidated(FlowMessageReadonly $flowMessage): void
{
    $data = $flowMessage->getMessage()->getRawData();  // array<string, mixed> der Properties
    $flowMessage->getHash();            // eindeutiger Hash dieser FlowMessage → Idempotenz-Key
    $flowMessage->getFlowHash();        // Flow-Instanz-Hash (gleich für alle Messages eines Flows)
    $flowMessage->getFlowType();        // z.B. 'flow.order.v1'
    $flowMessage->getMessageSource();   // FQCN der Original-Message
    $flowMessage->getTime();            // DateTimeImmutable
}
```

`getMessage()` liefert eine `ReadonlyMessage`; `getRawData(): array` enthält die serialisierten Properties (Keys = Property-Namen, DTOs als verschachtelte Arrays, Datumswerte als Strings). Typisierte Eigenschaften nicht annehmen — auf das Array zugreifen und Werte selbst casten.

`getMessageHash()` ist **nicht** eindeutig pro Message — es ist der Struktur-Hash der Message-Klasse. Nicht für Idempotenz verwenden.

## Schritt 4: Idempotenz (Pflicht)

Die Projection-Queue ist **at-least-once**. Eine Message kann mehrfach zugestellt werden, und ein Fehler in einer Methode acked die Message trotzdem (wird als `ProjectionException` persistiert, blockiert die Queue nicht, wird **nicht** erneut zugestellt). Handler-Methoden **müssen idempotent** sein:

- Upserts statt blinder Inserts (z.B. `INSERT ... ON DUPLICATE KEY UPDATE` / `ON CONFLICT`), Schlüssel aus fachlicher ID oder `getFlowHash()`
- Vor Side-Effects prüfen, ob bereits ausgeführt — Idempotenz-Key: `getHash()`
- Fehler, die nicht verloren gehen dürfen, selbst behandeln (Retry im Handler oder eigene Fehler-Tabelle)

Diesen Hinweis im generierten Code als Kommentar setzen.

## Schritt 5: Service-Abhängigkeiten (DI)

Handler werden über den Flowcrafter-eigenen Container instanziiert (nicht den einer Symfony-App). Service-Dependencies als `private readonly` Constructor-Parameter deklarieren — sie müssen in der `DependencyRegistry` (`flowcrafter.php`, `setDependencyRegistry()`) registriert sein:

```php
public function __construct(
    private readonly OrderReadModelRepository $repository,
) {}
```

Handler werden **pro Worker-Prozess einmal** gebaut und wiederverwendet — keinen Zustand zwischen Messages in Properties halten.

## Schritt 6: Projection-Klasse generieren

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Projections;

use App\Flowcrafter\Messages\{MessageSource};
use Wundii\Flowcrafter\Attribute\FlowProjection;
use Wundii\Flowcrafter\Attribute\FlowProjectionMessage;
use Wundii\Flowcrafter\FlowMessageReadonly;
use Wundii\Flowcrafter\Interface\ProjectionHandlerInterface;

#[FlowProjection('{flow-type}')]
class {ClassName}Projection implements ProjectionHandlerInterface
{
    // public function __construct(
    //     private readonly {SomeRepository} $repository,
    // ) {}

    #[FlowProjectionMessage({MessageSource}::class)]
    public function on{MessageShortName}(FlowMessageReadonly $flowMessage): void
    {
        $data = $flowMessage->getMessage()->getRawData();

        // TODO: Read Model upserten / Side-Effect auslösen.
        // Muss idempotent sein — die Queue ist at-least-once (Idempotenz-Key: $flowMessage->getHash()).
    }
}
```

Pro angegebenem Message-Source eine eigene `on{Name}()`-Methode erzeugen und die erwarteten Keys aus den Message-Properties im Code verwenden.

## Schritt 7: Output

1. Generierten Code mit Erklärung anzeigen
2. Den verwendeten Flow-Type-String gegen den Flow gegenprüfen (existiert er, stimmt die **aktuelle** Version?)
3. Prüfen ob die referenzierten Message-Klassen existieren und im Flow vorkommen
4. Datei schreiben (Write bzw. Edit bei bestehendem Handler)
5. Hinweis: Der Worker läuft als eigener Prozess — `vendor/bin/flowcrafter projection:worker` (bzw. im Dev-Modus als überwachter Subprozess). Handler werden automatisch entdeckt; ein laufender Worker muss nach dem Anlegen neu gestartet werden (im `dev`-Modus automatisch). Für Tests siehe `../flowcrafter/references/testing.md` → „Projection-Handler testen“
