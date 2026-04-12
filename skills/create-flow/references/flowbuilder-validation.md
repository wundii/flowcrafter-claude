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

**Prüfung**: `FlowBuilder::collectAllMessages()` — mindestens ein Stub muss die Init-Message als Constructor-Parameter haben.

**Fehlermeldung**: `MessageInit "X" is not added to the flow.`

**Fix**: Einen Stub hinzufügen dessen Constructor einen Parameter vom Typ der Init-Message hat.

## 3. Return-Message in returnTypes() vorhanden

**Prüfung**: Die in `FlowBuilder` deklarierte Return-Message muss in `returnTypes()` mindestens eines Stubs auftauchen.

**Fehlermeldung**: `MessageReturn "X" is not added to the flow.`

**Fix**: Sicherstellen dass ein Stub `[{ReturnMessage}::class]` in `returnTypes()` zurückgibt.

## 4. Keine Duplikate

**Fehlermeldung**: `Stub "X" is already added to the flow.`

**Fix**: Doppelten `addStub()`-Aufruf entfernen.

## 5. Kein Zyklus (DFS-Erkennung)

**Erkennung**: Tiefensuche (DFS) auf dem Stub-Dependency-Graph. `white` = unbesucht, `gray` = im Stack, `black` = fertig.

**Fehlermeldung**: `Loop detected in stub chain: StubA -> StubB -> StubA`

**Fix**: Message-Kette umstrukturieren. Kein Stub darf (direkt oder indirekt) eine Message produzieren die er selbst konsumiert.

## 6. Alle Stubs erreichbar (BFS)

**Erkennung**: BFS ab dem Stub der die Init-Message konsumiert. Alle anderen Stubs müssen über Message-Ketten erreichbar sein.

**Fehlermeldung**: `The following stubs are not connected to the flow: StubX`

**Fix**: Sicherstellen dass StubX eine Message konsumiert die ein anderer Stub im Flow produziert. Andernfalls den Stub aus dem Flow entfernen.

## 7. Keine hängenden MessageDataInterface-Return-Types

**Prüfung**: Für jeden Stub wird `returnTypes()` überprüft. Jeder Return-Type der `MessageDataInterface` implementiert (aber NICHT `MessageReturnInterface`) muss als Constructor-Parameter-Typ eines anderen Stubs vorkommen.

**Fehlermeldung**: `Stub "X" produces message "Y" that is not consumed by any stub.`

**Fix**: Entweder einen downstream Stub hinzufügen der `Y` konsumiert, oder die Message auf `MessageReturnInterface` ändern falls es der terminale Output ist.

## Fehlertabelle

| Fehler | Ursache | Fix |
|---|---|---|
| Type-Format ungültig | Falsches Format | `flow.<name>.v<N>` verwenden |
| Init-Message nicht konsumiert | Kein Stub nimmt die Init-Message | Stub mit Init-Message als Constructor-Param hinzufügen |
| Return-Message fehlt | Kein Stub gibt die Return-Message zurück | In `returnTypes()` eines Stubs eintragen |
| Duplikat-Stub | Gleiche Klasse zweimal `addStub()` | Duplikat entfernen |
| Zyklus erkannt | Ringabhängigkeit zwischen Stubs | Message-Kette umstrukturieren |
| Stub nicht erreichbar | Stub hat keine Verbindung zum Init-Stub | Message-Routing korrigieren oder Stub entfernen |
| Hängende MessageDataInterface | Produzierte Message wird nicht konsumiert | Downstream Stub hinzufügen oder MessageReturnInterface nutzen |
