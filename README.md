# flowcrafter-claude

Claude Code Plugin für die [Flowcrafter](https://github.com/wundii/flowcrafter) PHP Workflow-Engine. Hilft dabei, schneller und besser Flows, Stubs, Messages und Schedules zu erstellen.

## Installation

```bash
/install-plugin https://github.com/wundii/flowcrafter-claude
```

Nach der Installation steht das Plugin in jedem Projekt automatisch zur Verfügung.

## Skills

### Automatisch (model-invoked)

Der `flowcrafter`-Skill wird automatisch aktiviert, sobald Flowcrafter-Begriffe im Gespräch fallen (`FlowBuilder`, `StubInterface`, `*Flow.php` etc.). Claude erhält dann Framework-Wissen, korrekte Namespaces und Code-Defaults — ohne dass ein Befehl eingegeben werden muss.

### Slash-Commands (user-invoked)

| Command | Beschreibung | Beispiel |
|---|---|---|
| `/create-flow` | Flow-Klasse mit FlowBuilder-DSL generieren | `/create-flow WeatherComfort CityRequestMessage WeatherReportMessage` |
| `/create-stub` | Stub-Klasse mit Message-Injection generieren | `/create-stub FetchWeather` |
| `/create-message` | Message-Klasse (init/data/return) generieren | `/create-message CityRequest init city:string` |
| `/create-schedule` | Schedule-Klasse mit Cron-Ausdruck generieren | `/create-schedule WeatherComfort WeatherComfortFlow "0 * * * *"` |
| `/analyze-flow` | Flow auf Fehler und Verbesserungen prüfen | `/analyze-flow WeatherComfortFlow` |

## Verwendung

### Neuen Flow aufbauen (typischer Workflow)

**1. Messages definieren:**
```
/create-message OrderRequest init orderId:string customerId:string
/create-message OrderProcessed return orderId:string status:string
```

**2. Stubs erstellen:**
```
/create-stub ValidateOrder
/create-stub ProcessPayment
/create-stub SendConfirmation
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
