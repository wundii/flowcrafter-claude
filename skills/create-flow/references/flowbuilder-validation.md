# FlowBuilder Validierungsregeln

Alle Fehler werfen `InvalidArgumentException`. Regeln 1–3 greifen bereits im `FlowBuilder`-Constructor bzw. in `addStep()`, die übrigen bei `build()` (in dieser Reihenfolge).

## 1. Type-String-Format (Constructor)

**Regex**: `/^flow\..+\.v\d+$/`

| Gültig | Ungültig |
|---|---|
| `flow.order-processing.v1` | `OrderFlow` |
| `flow.weather-comfort.v2` | `flow_order_v1` |
| `flow.home.energy.system.v1` | `flow.order` (kein `.v<N>`) |
| | `flow.order.V1` (großes V) |

**Fehlermeldung**: `Flow type "X" must start with "flow." and end with ".v" followed by a number (e.g. "flow.example.v1").`

**Fix**: Type-String auf `flow.<kebab-case-name>.v1` korrigieren.

## 2. Init-/Return-Klassen (Constructor)

Init-Message muss `MessageInitInterface`, Return-Message (falls angegeben) `MessageReturnInterface` implementieren.

**Fehlermeldung**: `Message must be an instance of MessageInitInterface` / `... MessageReturnInterface`

## 3. Step-Klasse und Duplikate (addStep)

- Klasse muss `StepInterface` implementieren: `Step must be an instance of StepInterface`
- Klasse muss einen Constructor haben: `The source class must have a constructor.`
- Constructor-Message-Parameter müssen ein Message-Interface implementieren: `Message "X" does not implement any known message interface.`
- Kein Duplikat: `Step "X" is already added to the flow.` → doppelten `addStep()`-Aufruf entfernen

## 4. Init-Message konsumiert (build)

**Prüfung**: mindestens ein Step hat die Init-Message als Constructor-Parameter.

**Fehlermeldung**: `MessageInit "X" is not added to the flow.`

**Fix**: Einen Step hinzufügen dessen Constructor einen Parameter vom Typ der Init-Message hat.

## 5. Return-Message produziert (build)

**Prüfung**: Die im `FlowBuilder` deklarierte Return-Message muss in `returnTypes()` mindestens eines Steps auftauchen.

**Fehlermeldung**: `MessageReturn "X" is not added to the flow.`

**Fix**: Sicherstellen dass ein Step `[{ReturnMessage}::class]` in `returnTypes()` zurückgibt. Häufige Ursache: `returnTypes()` greift auf `$this` zu und liefert deshalb bei der Schema-Erzeugung (Instanz ohne Constructor) nichts — `returnTypes()` muss ein konstantes Array sein.

## 6. Kein Zyklus (DFS)

**Erkennung**: Tiefensuche auf dem Step-Graph (`white`/`gray`/`black`).

**Fehlermeldung**: `Loop detected in step chain: StepA -> StepB -> StepA`

**Fix**: Message-Kette umstrukturieren. Kein Step darf (direkt oder indirekt) eine Message produzieren die er selbst konsumiert.

## 7. Alle Steps erreichbar (BFS)

**Erkennung**: BFS ab dem ersten Step, der die Init-Message konsumiert. Alle anderen Steps müssen über Message-Ketten erreichbar sein.

**Fehlermeldung**: `The following steps are not connected to the flow: StepX`

**Fix**: Sicherstellen dass StepX eine Message konsumiert die ein anderer Step im Flow produziert. Andernfalls den Step aus dem Flow entfernen.

## 8. Keine hängenden DataMessages

**Prüfung**: Jeder Return-Type in irgendeinem `returnTypes()`, der **nicht** `MessageReturnInterface` implementiert, muss als Constructor-Parameter-Typ eines Steps vorkommen. Es gibt **keine** Ausnahme für Seitenzweige.

**Fehlermeldung**: `Step "X" produces message "Y" that is not consumed by any step.`

**Fix**:
- Seitenzweig/Seiteneffekt → Step gibt `bool` zurück (`returnTypes()` = `[]`) statt einer Message
- Fehlender Konsument → downstream Step hinzufügen der `Y` konsumiert
- `Y` ist das eigentliche Flow-Ergebnis → `Y` auf `MessageReturnInterface` umstellen und im `FlowBuilder` als Return-Message deklarieren

## Nicht validiert — trotzdem Pflicht

| Regel | Warum |
|---|---|
| Ein Produzent pro Message-Typ | Messages werden typbasiert geroutet und injiziert; zwei Produzenten führen zu nicht-deterministischem Verhalten |
| Nur ein Step liefert eine Return-Message | „Erste Return-Message gewinnt“ — abhängig von der Ausführungsreihenfolge |
| `returnTypes()` passt zu `process()` | Liefert `process()` einen nicht deklarierten Typ, fehlt er im Graph (keine Downstream-Steps, falsche Validierung) |
| Type-String projektweit eindeutig | Storage, Queue und Projections adressieren Flows über den Type-String |

## Schema-Hash und Versionierung

`retries`, `delay`, `runOnce`, die Step-Struktur und die Message-Hashes sind Teil des Schema-Hashs (`FlowSchema::getHash()`). Eine Änderung ergibt einen neuen Hash; bestehende Instanzen mit altem Hash sind dann nicht mehr ausführbar (`Flow is not executable, because the flowSchemaHash is different from the stored version`).

**Fix**: Flow-Version erhöhen (`v1` → `v2`) und Referenzen auf den alten Type-String (Projections, Tests) anpassen.

## Fehlertabelle

| Fehler | Ursache | Fix |
|---|---|---|
| Type-Format ungültig | Falsches Format | `flow.<name>.v<N>` verwenden |
| Init/Return falsches Interface | Message-Klasse implementiert falsches Interface | Interface korrigieren |
| Step ohne Constructor | Step hat keinen `__construct` | Constructor mit konsumierter Message hinzufügen |
| Duplikat-Step | Gleiche Klasse zweimal `addStep()` | Duplikat entfernen |
| Init-Message nicht konsumiert | Kein Step nimmt die Init-Message | Step mit Init-Message als Constructor-Param hinzufügen |
| Return-Message fehlt | Kein Step gibt die Return-Message zurück | In `returnTypes()` eines Steps eintragen (konstantes Array) |
| Zyklus erkannt | Ringabhängigkeit zwischen Steps | Message-Kette umstrukturieren |
| Step nicht erreichbar | Step hat keine Verbindung zum Init-Step | Message-Routing korrigieren oder Step entfernen |
| Hängende DataMessage | Produzierte Message wird nicht konsumiert | `bool` zurückgeben, Konsument hinzufügen oder auf `MessageReturnInterface` umstellen |
| Flow not executable | Schema geändert ohne Versions-Bump | Flow-Version erhöhen (`v1` → `v2`) |
