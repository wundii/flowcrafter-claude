# flowcrafter-claude

Claude Code Plugin für die [Flowcrafter](https://github.com/wundii/flowcrafter) PHP Workflow-Engine. Hilft dabei, schneller und besser Flows, Steps, Messages und Schedules zu erstellen.

## Installation

Die Installation erfolgt in zwei Schritten in Claude Code:

**1. Marketplace registrieren:**
```
/plugin marketplace add wundii/flowcrafter-claude
```

**2. Plugin installieren:**
```
/plugin install flowcrafter@flowcrafter-claude
```

Beim Installieren wählt Claude Code den Scope:
- **user** — steht in allen Projekten zur Verfügung (empfohlen)
- **project** — nur im aktuellen Projekt (`.claude/settings.json`)
- **local** — nur lokal, nicht committed

### Alternativ: Interaktiver Plugin-Manager

```
/plugin
```

Im Tab **Discover** das Plugin suchen und per Enter installieren.

## Skills

### Automatisch (model-invoked)

Der `flowcrafter`-Skill wird automatisch aktiviert, sobald Flowcrafter-Begriffe im Gespräch fallen (`FlowBuilder`, `StepInterface`, `*Flow.php` etc.). Claude erhält dann Framework-Wissen, korrekte Namespaces und Code-Defaults — ohne dass ein Befehl eingegeben werden muss.

### Slash-Commands (user-invoked)

| Command | Beschreibung | Beispiel |
|---|---|---|
| `/create-flow` | Flow-Klasse mit FlowBuilder-DSL generieren | `/create-flow WeatherComfort CityRequestMessage WeatherReportMessage` |
| `/create-step` | Step-Klasse mit Message-Injection generieren | `/create-step FetchWeather` |
| `/create-message` | Message-Klasse (init/data/return) generieren | `/create-message CityRequest init city:string` |
| `/create-schedule` | Schedule-Klasse mit Cron-Ausdruck generieren | `/create-schedule WeatherComfort WeatherComfortFlow "0 * * * *"` |
| `/create-projection` | Projection-Handler (Read Model / async Side-Effect) generieren | `/create-projection Order flow.order.v1 OrderValidatedMessage` |
| `/analyze-flow` | Flow auf Fehler und Verbesserungen prüfen | `/analyze-flow WeatherComfortFlow` |

## Verwendung

### Neuen Flow aufbauen (typischer Workflow)

**1. Messages definieren:**
```
/create-message OrderRequest init orderId:string customerId:string
/create-message OrderProcessed return orderId:string status:string
```

**2. Steps erstellen:**
```
/create-step ValidateOrder
/create-step ProcessPayment
/create-step SendConfirmation
```

**3. Flow zusammensetzen:**
```
/create-flow OrderProcessing OrderRequestMessage OrderProcessedMessage
```

**4. Flow prüfen:**
```
/analyze-flow OrderProcessingFlow
```

### Flow ohne externen Input (EmptyInitMessage)

Falls ein Flow keinen externen Input braucht (z.B. scheduler-getriggert), einfach beschreiben — Claude verwendet automatisch die eingebaute `EmptyInitMessage`:

```
/create-flow DailyReport
```

### Schedule hinzufügen

```
/create-schedule DailyReport DailyReportFlow "0 8 * * *"
```

### Bestehenden Code analysieren

```
/analyze-flow --all
```

Analysiert alle `*Flow.php`-Dateien im Projekt und listet Probleme sowie Verbesserungsvorschläge auf.

## Anforderungen

- [Claude Code](https://claude.ai/code) CLI
- PHP-Projekt mit `wundii/flowcrafter`

## Lizenz

MIT
