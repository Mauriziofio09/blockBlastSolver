# Projekt: Block-Blast-Solver (iPhone-PWA)

Baue eine offline-fähige Web-App (PWA, mobile-first, optimiert für iPhone/Safari), die mir für das Handyspiel „Block Blast" die besten nächsten Züge anzeigt.
Ich gebe das Board und meine verfügbaren Steine manuell in der App ein. Es gibt keine Screenshot-Erkennung.
Ziel: möglichst lange überleben und dabei durchgehend Combos halten. Bei Konflikt hat Überleben Vorrang (lieber Combo brechen als Game Over). Punkte sind zweitrangig.

## Spielregeln (so modellieren)
- Spielfeld 8x8.
- Pro Runde 3 Steine (Polyominos, nur orthogonal verbundene Felder). Steine können NICHT gedreht werden. Jede Ausrichtung ist ein eigener Stein im Katalog.
- Stein-Katalog als Datendatei (`src/core/pieces.ts`), jeder Stein mit ID, Form und Gewicht für die Zufallsverteilung. Mindestens:
  1x1, Linien 1x2–1x5 horizontal und vertikal, 2x2, 3x3, 2x3 und 3x2, kleine L (3 Felder, 4 Ausrichtungen), L/J (4 Felder, 8 Ausrichtungen), grosse L (5 Felder, 3x3-Ecke, 4 Ausrichtungen), T (4 Ausrichtungen), S/Z (4 Ausrichtungen).
- Alle 3 Steine werden in beliebiger Reihenfolge platziert, danach kommen 3 neue.
- Nach jeder Platzierung werden volle Reihen UND volle Spalten gleichzeitig gelöscht (ein Feld im Schnittpunkt zählt nur einmal).
- Combo: Jede Platzierung mit mindestens einem Clear erhöht den Combo-Zähler. Nach `comboGrace` Platzierungen ohne Clear bricht die Combo (konfigurierbar, Default: siehe offene Punkte).
- Game Over, wenn ein verbleibender Stein nirgends passt.
- Solver-Input ist der vollständige Zustand: Board, aktueller Combo-Zähler, Platzierungen seit letztem Clear und 1–3 verbleibende Steine (falls ich schon einen gelegt habe).

## Architektur
- `src/core/` – reine TypeScript-Logik ohne DOM: `board.ts`, `pieces.ts`, `solver.ts`, `evaluate.ts`, `montecarlo.ts`, `rng.ts`
- `src/worker/` – Web Worker, der den Solver ausführt (abbrechbar)
- `src/ui/` – Oberfläche
- `src/config/weights.ts` – alle Gewichte und Parameter an einem Ort, kommentiert

**Bitboard von Anfang an (Meilenstein 1):** Das 8x8-Feld als zwei `Uint32` (lo/hi), kein BigInt (zu langsam).
Vorberechnen:
- für jeden Stein alle gültigen Positions-Masken,
- Masken für alle 8 Reihen und 8 Spalten.

Kollision, Platzierung und Clear-Erkennung laufen nur über Bit-Operationen.

## Solver
1. **Enumeration:** alle Reihenfolgen (bei identischen Steinen dedupliziert) × alle gültigen Positionen, mit Clears zwischen den Platzierungen. Identische Endzustände (Board + Combo-Zustand) per Hash deduplizieren.
2. **Wenn nicht alle Steine passen:** bestmöglichen Teilzug zurückgeben und klar warnen.
3. **Bewertung** (alle Gewichte in `weights.ts`):
   - gelöschte Linien und Combo-Fortsetzung (stark gewichtet), Bonus für Multi-Line-Clears,
   - Combo-Potenzial für die nächste Runde: Anzahl Reihen/Spalten mit 6–7 von 8 belegt,
   - Strafen für isolierte Einzellöcher, fragmentierte Leerflächen (Anzahl/Grösse zusammenhängender Leerregionen) und Anzahl belegter Felder,
   - Überlebenscheck: Passen die gefährlichen Steine (3x3, 1x5, 5x1, grosse L) danach noch irgendwo? Wenn nein, starke Strafe.
4. **Monte-Carlo-Lookahead:**
   - Für die Top-N Kandidaten (Default 20) je M zufällige 3er-Sets (Default 200) aus der gewichteten Verteilung ziehen, mit seeded RNG.
   - Pro Set NICHT den vollen Brute-Force-Solver laufen lassen, sondern eine billige Rollout-Policy:
     - zuerst ein Feasibility-Check mit Early Exit (passen alle 3 Steine in irgendeiner Reihenfolge?),
     - dann eine Runde greedy mit der Heuristik spielen.
   - Finaler Score = gewichtete Kombination aus Überlebensrate, erwarteten Clears und Combo-Erhalt.
5. **Zeitbudget (Anytime):**
   - Das Heuristik-Ergebnis sofort anzeigen (Ziel < 200 ms).
   - Danach verfeinert der Worker per Monte Carlo, bis das Budget abläuft (Default 1500 ms, konfigurierbar), und aktualisiert das Ergebnis.
   - Rechenzeit und Anzahl Simulationen in einem Debug-Overlay anzeigen.
6. **Determinismus:** gleicher Seed → gleiches Ergebnis (für Tests).

## Input (nur manuell)
- **Board-Editor:**
  - 8x8-Grid, Felder per Tap umschalten. Wischen über mehrere Felder füllt bzw. leert sie.
  - Wird nur beim Start oder zum Korrigieren gebraucht.
  - Buttons: „Neues Spiel (leeres Board)" und „Board bearbeiten".
- **Board-Tracking:** Nach „Zug ausgeführt" übernimmt die App das resultierende Board und den Combo-Zustand automatisch. Habe ich im Spiel anders gelegt als vorgeschlagen, kann ich das Board jederzeit per Tap korrigieren.
- **Stein-Picker (wichtigster Input, muss in ~3 Sekunden bedienbar sein):**
  - Alle Steine aus dem Katalog als Mini-Vorschauen in einem scrollbaren Raster, gruppiert nach Grösse bzw. Typ, gross genug zum Antippen.
  - 3 Slots oben. Ein Tap auf einen Stein füllt den nächsten freien Slot, ein Tap auf einen Slot leert ihn. Derselbe Stein darf mehrfach gewählt werden.
  - Sind alle 3 Slots gefüllt, startet der Solver automatisch.
  - Mit 1–2 gefüllten Slots kann ich den Solver per Button „Berechnen" starten (für den Fall, dass ich schon einen Stein gelegt habe).
  - „Zuletzt verwendet"-Reihe mit den häufigsten Steinen ganz oben.
- **Eigene Formen:** kleiner 5x5-Editor. Gespeicherte eigene Formen erscheinen im Picker.

## Anzeige
- Der beste Zug als 3 nummerierte Schritte. Jeder Schritt zeigt:
  - das Board im Zustand VOR diesem Schritt (also nach den Clears der vorherigen Schritte),
  - den Stein halbtransparent an der Zielposition mit seiner Nummer,
  - die Linien, die dabei gelöscht werden (aufblinkend).
- Darunter:
  - Combo-Zähler-Vorschau,
  - geschätzte Überlebenswahrscheinlichkeit (mit Anzahl Simulationen),
  - 2 alternative Züge zum Aufklappen, mit Score-Differenz zum besten Zug.
- Button „Zug ausgeführt": übernimmt das resultierende Board und den Combo-Zustand, leert die Slots und springt zurück zum Stein-Picker.
- Button „Rückgängig" (mehrstufig).
- Der Combo-Zähler läuft über die Runden mit und ist immer sichtbar.
- Der gesamte Zustand (Board, Combo, Verlauf) wird in localStorage gespeichert, damit beim Schliessen der App nichts verloren geht.
- UI-Sprache: Deutsch.

## Technik
- Vanilla TypeScript (strict) + Vite, kein Framework.
- `vite-plugin-pwa` für Manifest und Service Worker. Komplett offline-fähig, keine Backend-Abhängigkeit.
- Unit-Tests mit Vitest, Benchmarks mit `vitest bench`.
- iOS-Details:
  - `apple-touch-icon` und `apple-mobile-web-app-capable`,
  - Safe-Area-Insets beachten,
  - keine Hover-Abhängigkeiten,
  - Touch-Ziele mindestens 44 pt.
- Dunkles, cleanes Design.
- Optional zum Schluss: statischer Build als Docker-Container (nginx) für meinen Hetzner-Server, als Subdomain von battle-isle.app.

## Meilensteine (mit Abnahmekriterien)
1. **Core + Solver + Tests**
   - Umfang: Bitboard, Clears, Combo-Logik, Enumeration, Heuristik.
   - Mindestens diese Test-Boards mit offensichtlichem Zug:
     - eine Reihe mit 7/8 belegt + ein 1x1 → der Stein füllt die Lücke,
     - Reihe und Spalte gleichzeitig löschen,
     - Reihenfolge entscheidend: Stein A passt nur, wenn B vorher eine Linie löscht,
     - Platz für den 3x3 erhalten statt kurzfristigem Clear,
     - drei identische Steine,
     - nicht alle Steine passen → Teilzug + Warnung.
   - Dazu ein Benchmark auf einem halbvollen Board.
2. **Board-Editor, Stein-Picker + Anzeige der Züge**
   - Umfang: Editor, Picker und Zug-Anzeige, mit dem Heuristik-Solver im Worker.
   - Abnahme: Ich kann eine echte Spielrunde durchspielen und pro Runde nur 3 Steine eingeben.
3. **Monte-Carlo-Lookahead + Performance-Optimierung**
   - Ziel: Ergebnis < 1,5 s auf dem iPhone, gemessen im Debug-Overlay.
4. **PWA-Feinschliff**
   - Umfang: installierbar, offline, Icons, Persistenz, optional das Deployment.

## Arbeitsweise
- **Vor dem Start:** Stell mir deine Rückfragen gesammelt in einer Nachricht. Zeig mir dann einen kurzen Plan (Dateistruktur + zentrale Datentypen) und warte auf mein OK.
- **Nach jedem Meilenstein:** Stopp und zeig mir:
  - was du gebaut hast,
  - wie ich es teste (Befehle),
  - die Testresultate,
  - bekannte Schwächen.
- Ein Git-Commit pro Meilenstein.
- Triff keine stillen Annahmen: Jede Annahme kommt mit Begründung in `DECISIONS.md`.

## Offene Punkte (bitte mit mir klären)
- Combo-Regel: Wie viele Platzierungen ohne Clear toleriert das Spiel, bevor die Combo bricht?
- Fehlen Formen im Katalog?
- Stein-Verteilung: gleichverteilt oder gewichtet? Gibt das Spiel bevorzugt Steine, die passen?
- Gewichtung Überleben vs. Combo: Ist „Überleben vor Combo" richtig?
