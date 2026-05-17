# FlowBuilder Validierungsregeln

Alle Validierungen werden bei `FlowBuilder::build()` ausgeführt. Fehler werfen `InvalidArgumentException`.

## 1. Type-String-Format

**Regex**: `/^flow\..+\.v\d+$/`

| Gültig | Ungültig |
|---|---|
| `flow.order-processing.v1` | `OrderFlow` |
| `flow.weather-comfort.v2` | `flow_order_v1` |
| `flow.home.energy.system.v1` | `flow.order` (kein `.v<N>`) |
| | `flow. name.v1` (Leerzeichen) |

**Fehlermeldung**: `Flow type "X" must start with "flow." and end with ".v" followed by a number`

**Fix**: Type-String auf `flow.<kebab-case-name>.v1` korrigieren.

## 2. Init-Message konsumiert

**Prüfung**: `FlowBuilder::collectAllMessages()` — mindestens ein Step muss die Init-Message als Constructor-Parameter haben.

**Fehlermeldung**: `MessageInit "X" is not added to the flow.`

**Fix**: Einen Step hinzufügen dessen Constructor einen Parameter vom Typ der Init-Message hat.

## 3. Return-Message in returnTypes() vorhanden

**Prüfung**: Die in `FlowBuilder` deklarierte Return-Message muss in `returnTypes()` mindestens eines Steps auftauchen.

**Fehlermeldung**: `MessageReturn "X" is not added to the flow.`

**Fix**: Sicherstellen dass ein Step `[{ReturnMessage}::class]` in `returnTypes()` zurückgibt.

## 4. Keine Duplikate

**Fehlermeldung**: `Step "X" is already added to the flow.`

**Fix**: Doppelten `addStep()`-Aufruf entfernen.

## 5. Kein Zyklus (DFS-Erkennung)

**Erkennung**: Tiefensuche (DFS) auf dem Step-Dependency-Graph. `white` = unbesucht, `gray` = im Stack, `black` = fertig.

**Fehlermeldung**: `Loop detected in step chain: StepA -> StepB -> StepA`

**Fix**: Message-Kette umstrukturieren. Kein Step darf (direkt oder indirekt) eine Message produzieren die er selbst konsumiert.

## 6. Alle Steps erreichbar (BFS)

**Erkennung**: BFS ab dem Step der die Init-Message konsumiert. Alle anderen Steps müssen über Message-Ketten erreichbar sein.

**Fehlermeldung**: `The following steps are not connected to the flow: StepX`

**Fix**: Sicherstellen dass StepX eine Message konsumiert die ein anderer Step im Flow produziert. Andernfalls den Step aus dem Flow entfernen.

## 7. Keine hängenden MessageDataInterface-Return-Types

**Prüfung**: Für jeden Step wird `returnTypes()` überprüft. Jeder Return-Type der `MessageDataInterface` implementiert (aber NICHT `MessageReturnInterface`) muss als Constructor-Parameter-Typ eines anderen Steps vorkommen.

**Fehlermeldung**: `Step "X" produces message "Y" that is not consumed by any step.`

**Fix**: Entweder einen downstream Step hinzufügen der `Y` konsumiert, oder die Message auf `MessageReturnInterface` ändern falls es der terminale Output ist.

## 8. Schema-Hash und Retry-Parameter

`retries` und `delay` sind Teil des Schema-Hashs (`FlowSchema::getHash()`). Eine Änderung dieser Werte — auch ohne sonstige strukturelle Änderungen — ergibt einen neuen Hash. Bestehende Flow-Instanzen mit dem alten Hash werden beim Re-Run als inkompatibel erkannt.

**Fix**: Flow-Version erhöhen (z.B. `v1` → `v2`) wenn `retries` oder `delay` geändert werden.

## Fehlertabelle

| Fehler | Ursache | Fix |
|---|---|---|
| Type-Format ungültig | Falsches Format | `flow.<name>.v<N>` verwenden |
| Init-Message nicht konsumiert | Kein Step nimmt die Init-Message | Step mit Init-Message als Constructor-Param hinzufügen |
| Return-Message fehlt | Kein Step gibt die Return-Message zurück | In `returnTypes()` eines Steps eintragen |
| Duplikat-Step | Gleiche Klasse zweimal `addStep()` | Duplikat entfernen |
| Zyklus erkannt | Ringabhängigkeit zwischen Steps | Message-Kette umstrukturieren |
| Step nicht erreichbar | Step hat keine Verbindung zum Init-Step | Message-Routing korrigieren oder Step entfernen |
| Hängende MessageDataInterface | Produzierte Message wird nicht konsumiert | Downstream Step hinzufügen oder MessageReturnInterface nutzen |
| Schema-Hash-Mismatch | `retries`/`delay` geändert ohne Versions-Bump | Flow-Version erhöhen (`v1` → `v2`) |
