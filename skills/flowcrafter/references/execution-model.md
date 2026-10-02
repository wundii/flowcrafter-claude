# Flowcrafter — Execution-Modell

## Rekursive Ausführung

Der `FlowRunner` arbeitet **rekursiv message-driven** (depth-first), nicht linear Step für Step:

1. Eine Message wird produziert (Start: InitMessage)
2. Der Runner schlägt in der `messageToStepsMap` nach, welche Steps diese Message konsumieren (in `addStep()`-Reihenfolge)
3. Für jeden konsumierenden Step: prüfe ob **alle** benötigten Messages verfügbar sind (`executableMessages`)
4. Wenn ja → Step ausführen → Ergebnis verarbeiten (siehe Step Return-Types)
5. Bei `MessageDataInterface`-Return → **sofortiger rekursiver Aufruf** mit der neuen Message (zurück zu Schritt 2), danach erst der nächste Konsument aus Schritt 2

Folge der Depth-First-Rekursion: Die Reihenfolge, in der Seitenzweige und Hauptkette laufen, hängt von der `addStep()`-Reihenfolge ab. Logik darf sich nicht darauf verlassen.

## Branching — Mehrere Steps konsumieren dieselbe Message

Wenn mehrere Steps dieselbe Message als Constructor-Parameter deklarieren, werden **alle** getriggert wenn diese Message produziert wird:

```
m1 → s1 → m2 → s2 → m3 (Hauptkette)
              → s3 → bool (Seitenzweig, terminal)
```

Beide Steps (s2, s3) konsumieren m2. s2 führt die Hauptkette fort, s3 ist ein Seitenzweig der mit `bool` endet.

## Convergence — Step wartet auf Messages aus verschiedenen Branches

Ein Step kann Messages aus **verschiedenen Branches** benötigen. Er wird erst ausgeführt wenn **alle** seine Message-Dependencies verfügbar sind:

```
m1 → s1 → m2 → s2 → m3 → s4 → m4 ─┐
              → s3 → m6 ────────────┤
                                     └→ s5(m4+m6) → m5 (Return)
```

s5 benötigt sowohl m4 (aus dem Hauptzweig) als auch m6 (aus dem anderen Zweig). Der Runner führt s5 erst aus, wenn beide Messages produziert wurden. m6 ist hier eine DataMessage und wird von s5 konsumiert — das ist erlaubt. Eine DataMessage, die **niemand** konsumiert, ist dagegen nie erlaubt.

## Fehlerverhalten

- Wirft `process()` eine Exception, wird bei `retries > 0` mit neuer Step-Instanz erneut versucht (blockierender `delay`).
- Nach dem letzten Fehlversuch: `FlowException` wird gespeichert, der Flow persistiert und die Exception **weitergeworfen**. Der gesamte Run endet — auch noch nicht ausgeführte Steps anderer Zweige laufen nicht mehr. Status: `FAILED`.
- Seitenzweige, deren Fehlschlag den Flow nicht abbrechen soll (z.B. Benachrichtigungen), fangen die Exception selbst und geben `false` zurück (Status `WARNING`).

## Flow-Status

| Status | Bedingung |
|---|---|
| `IN_PROGRESS` / `IN_PROGRESS_EXCEEDED` | Leaf-Steps noch nicht erreicht (nach > 1 h: EXCEEDED) |
| `OK` | Alle Leaf-Steps erreicht, alle bool-Results `true` |
| `WARNING` | Abgeschlossen, mindestens ein bool-Result `false` |
| `FAILED` | Exception im letzten Run |

## Wichtige Regeln

- **Erster Return gewinnt**: Nur die erste `MessageReturnInterface` wird als Flow-Ergebnis behalten, weitere werden ignoriert. Deshalb sollte nur die Hauptkette eine Return-Message produzieren.
- **Terminal ohne Fehler zur Laufzeit**: Hat eine Message keinen konsumierenden Step, kehrt die Rekursion einfach zurück. (Statisch verhindert `FlowBuilder::build()` aber unkonsumierte DataMessages.)
- **Keine Duplikat-Ausführung**: Jede Step+Message-Kombination wird pro Run maximal einmal ausgeführt.
- **Keine doppelten Produzenten**: Zwei Steps dürfen nicht denselben Message-Typ produzieren — Messages werden typbasiert geroutet und im Container per Klasse abgelegt. **Wird vom `FlowBuilder` nicht validiert.**
- **`runOnce`**: Bei Re-Runs einer bestehenden Instanz (mit Storage + `flowHash`) wird ein `runOnce`-Step übersprungen und sein gespeichertes Ergebnis weitergereicht.
- **`includeSteps`**: Beschränkt einen Run auf die genannten Steps plus alle nachgelagerten; fehlende Inputs werden aus dem früheren Run der Instanz übernommen (nur mit Storage).
