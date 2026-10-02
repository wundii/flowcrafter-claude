---
name: create-flow
description: >
  Generate or extend a Flowcrafter Flow class. Use when the user asks to
  "create a flow", "add a flow", "new flow", "define a workflow",
  "build a processing pipeline", "add a step to the flow", "bump the flow version",
  or wants to implement a FlowInterface class using the FlowBuilder DSL for
  Flowcrafter (wundii/flowcrafter).
argument-hint: <flow-name> [init-message-class] [return-message-class]
allowed-tools: Read, Glob, Grep, Write, Edit, Bash(php:*)
---

# Create Flowcrafter Flow

Generiere eine vollständige, validierte `FlowInterface`-Implementierung mit dem `FlowBuilder`-DSL — oder erweitere einen bestehenden Flow.

Konzepte, Validierungsregeln und Code-Defaults stehen im `flowcrafter`-Skill (`../flowcrafter/SKILL.md`). Bei Bedarf dort nachlesen, insbesondere „Step-Design-Leitfaden“, „Seitenzweige“ und „EmptyInitMessage“.

## Argumente

Der User hat aufgerufen mit: $ARGUMENTS

Parse als: `<flow-name>` (Pflicht), `[init-message-class]` (optional), `[return-message-class]` (optional). Falls Argumente fehlen, frage den User nach dem Zweck des Flows, der Init-Message und der Return-Message bevor du generierst. Braucht der Flow keinen externen Input → `EmptyInitMessage` statt einer eigenen Init-Message.

## Schritt 1: Projekt-Kontext erkennen

1. `composer.json` lesen → `wundii/flowcrafter` bestätigen, PHP-Version und PSR-4-Autoload ermitteln
2. Glob auf `src/**/*Flow.php` → bestehende Flows für Namespace und Directory-Konvention
3. Einen bestehenden Flow lesen um das genaue Muster zu bestätigen (Variable-Name `$flowBuilder`, Import-Pfade, Attribute)
4. Prüfen ob der Type-String bereits existiert: Grep nach `'flow.<name>.v` — Type-Strings müssen projektweit eindeutig sein

## Schritt 2: Klasse und Platzierung bestimmen

- **Klassenname**: PascalCase + `Flow`-Suffix (z.B. `order-processing` → `OrderProcessingFlow`)
- **Directory**: bestehendes Pattern; gibt es keins, `src/Flowcrafter/Flows/` vorschlagen
- **Namespace**: aus PSR-4 Autoload in `composer.json` ableiten
- **Type-String**: `flow.<kebab-case-name>.v1` (z.B. `flow.order-processing.v1`)

## Schritt 3: Message-Graph planen und prüfen (VOR Codegenerierung)

Bestehende Steps lesen (Constructor-Messages + `returnTypes()`) und den Graph skizzieren. Sicherstellen:

- **Type-String-Format** matcht `/^flow\..+\.v\d+$/`
- **Init-Message** wird von mindestens einem Step als Constructor-Parameter konsumiert
- **Return-Message** (optional) steht in `returnTypes()` eines Steps der **Hauptkette**
- **Jede DataMessage** wird von mindestens einem Step konsumiert — auch auf Seitenzweigen. Seitenzweige enden mit `bool`
- **Nur ein Produzent** pro Message-Typ und nur ein Step, der eine Return-Message liefert (vom FlowBuilder nicht validiert!)
- **Alle Steps erreichbar** vom Init-Step, **kein Zyklus**
- User warnen wenn referenzierte Steps oder Messages noch nicht existieren

Message-Graph dem User als ASCII-Diagramm zeigen.

## Schritt 4: Flow-Klasse generieren

```php
<?php

declare(strict_types=1);

namespace App\Flowcrafter\Flows;

use App\Flowcrafter\Messages\{InitMessageClass};
use App\Flowcrafter\Messages\{ReturnMessageClass};
use App\Flowcrafter\Steps\{StepClass};
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

        $flowBuilder->addStep({StepClass}::class);
        // Fehleranfälliger externer I/O (HTTP, DB, APIs):
        // $flowBuilder->addStep({StepClass}::class, retries: 3, delay: 500);
        // Seiteneffekt, der bei Re-Runs nicht wiederholt werden darf (Zahlung, Mail):
        // $flowBuilder->addStep({StepClass}::class, runOnce: true);

        return $flowBuilder->build();
    }
}
```

**Hinweise**:
- Variable heißt `$flowBuilder`
- `addStep()` nimmt den FQCN als `::class` Konstante
- `addStep()`-Reihenfolge = Ausführungsreihenfolge bei Branching (depth-first). Hauptkette in natürlicher Reihenfolge eintragen
- `retries` (default 0) / `delay` (default 200ms) nur für Steps mit externem I/O
- `runOnce: true` für Steps mit nicht wiederholbaren Seiteneffekten
- `retries`, `delay` und `runOnce` fließen in den Schema-Hash ein
- Optional `#[FlowGroup('group-name')]` — nur wenn der User es wünscht oder das Projekt es bereits verwendet
- Optional `#[FlowEphemeral(expiryDays: 14)]` — nur für kurzlebige Flows ohne Business-Daten (Health-Checks, Monitoring) und nur auf Wunsch. Cleanup läuft nur, wenn der Scheduler läuft
- **Nicht** `#[FlowSchedule]` auf den Flow setzen — dieses Attribut gehört auf die Schedule-Klasse

## Bestehenden Flow ändern

Wird ein bestehender Flow geändert (Step hinzu/weg, `retries`/`delay`/`runOnce`, Message-Properties):

1. Mit `Edit` ändern — nicht die Datei neu schreiben
2. Prüfen ob sich der Schema-Hash ändert (siehe `../flowcrafter/references/framework-concepts.md` → „Wann `v<N>` erhöhen“). Wenn ja: User fragen, ob die Version erhöht werden soll (`v1` → `v2`)
3. Bei Versions-Bump alle Referenzen auf den alten Type-String suchen (Grep `flow.<name>.v1`) — insbesondere `#[FlowProjection(...)]`-Handler und Tests — und dem User die nötigen Anpassungen zeigen. Projections abonnieren exakte Type-Strings und verpassen den neuen Flow sonst stillschweigend

## Schritt 5: Validieren

Wenn alle referenzierten Klassen existieren und `vendor/autoload.php` vorhanden ist, das Schema echt bauen lassen statt nur statisch zu prüfen:

```bash
php -r 'require "vendor/autoload.php"; echo \App\Flowcrafter\Flows\{ClassName}Flow::schema()->type(), PHP_EOL;'
```

Eine `InvalidArgumentException` entspricht einer der Regeln in `references/flowbuilder-validation.md` — Fix anhand der Tabelle dort vorschlagen. Schlägt der Aufruf aus Umgebungsgründen fehl (kein lokales PHP, Docker-Projekt), Validierung statisch durchführen und das dem User sagen.

## Schritt 6: Output

1. Generierten Code und Message-Graph mit Erklärung anzeigen
2. Datei schreiben (Write für neue Dateien, Edit für bestehende)
3. Fehlende Steps und Messages auflisten, `create-step` und `create-message` empfehlen
4. Optional: `create-test` für einen `FlowTestCase` vorschlagen

## Detaillierte Validierungsregeln

Siehe `references/flowbuilder-validation.md` für alle Fehlermeldungen und Fix-Vorschläge.
