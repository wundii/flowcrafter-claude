# Flowcrafter — Execution-Modell

## Rekursive Ausführung

Der `FlowRunner` arbeitet **rekursiv message-driven**, nicht linear Step für Step:

1. Eine Message wird produziert (Start: InitMessage)
2. Der Runner schlägt in der `messageToStepsMap` nach, welche Steps diese Message konsumieren
3. Für jeden konsumierenden Step: prüfe ob **alle** benötigten Messages verfügbar sind (`executableMessages`)
4. Wenn ja → Step ausführen → Ergebnis verarbeiten (siehe Step Return-Types)
5. Bei `MessageDataInterface`-Return → **rekursiver Aufruf** mit der neuen Message (zurück zu Schritt 2)

## Branching — Mehrere Steps konsumieren dieselbe Message

Wenn mehrere Steps dieselbe Message als Constructor-Parameter deklarieren, werden **alle** getriggert wenn diese Message produziert wird:

```
m1 → s1 → m2 → s2 → m3 (Hauptkette)
              → s3 → bool (Seitenzweig, terminal)
```

Beide Steps (s2, s3) konsumieren m2. s2 führt die Hauptkette fort, s3 ist ein Seitenzweig der mit bool endet.

## Convergence — Step wartet auf Messages aus verschiedenen Branches

Ein Step kann Messages aus **verschiedenen Branches** benötigen. Er wird erst ausgeführt wenn **alle** seine Message-Dependencies verfügbar sind:

```
m1 → s1 → m2 → s2 → m3 → s4 → m4 ─┐
              → s3 → m6 ────────────┤
                                     └→ s5(m4+m6) → m5 (Return)
```

s5 benötigt sowohl m4 (aus dem Hauptzweig) als auch m6 (aus dem Seitenzweig). Der Runner führt s5 erst aus, wenn beide Messages produziert wurden.

## Wichtige Regeln

- **Erster Return gewinnt**: Nur die erste `MessageReturnInterface` wird als Flow-Ergebnis behalten, weitere werden ignoriert
- **Terminal ohne Fehler**: Wenn eine Message keinen konsumierenden Step hat, kehrt die Rekursion einfach zurück — kein Fehler
- **Keine Duplikat-Ausführung**: Jede Step+Message-Kombination wird maximal einmal ausgeführt
- **Keine doppelten Produzenten**: Zwei Steps dürfen nicht denselben Message-Typ produzieren
