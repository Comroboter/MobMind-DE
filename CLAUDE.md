# MobMind-DE

Fork von https://github.com/Carlown/MobMind (MIT-Lizenz). Ziel: deutsche Version mit Groq als kostenloser KI, Mob-Smalltalk, artspezifischen Interaktionen und BlueMap-Anbindung.

## Arbeitsweise

- Wir arbeiten nach PLAN.md, Phase für Phase.
- Aktueller Stand und Erkenntnisse stehen in NOTES.md. Lege die Datei in Phase 0 an und halte sie nach jeder Phase aktuell.
- Nach jeder Phase: `./gradlew build` muss fehlerfrei laufen, wichtige Änderungen im Dev-Client testen, einen Commit machen und dann anhalten.
- `MobAiService.java` nicht komplett umbauen. Gezielt ändern, neue Logik lieber in neue Klassen auslagern.

## Sicherheit

- API-Keys niemals committen. Die Config-Datei mit dem Key gehört in die `.gitignore`.
- Das Repo ist öffentlich.
