# MobMind-DE: Arbeitsplan für Claude Code

## Kontext

- Fork von https://github.com/Carlown/MobMind (MIT-Lizenz, Credits an den Original-Autor im README behalten)
- Minecraft 26.2, Fabric Loader >= 0.19.3, Java 25
- Native sherpa-onnx-Bibliothek ist nur für win-x64 eingebunden
- Ziel: deutschsprachige Mobs, kostenlose KI, Smalltalk zwischen Mobs, artspezifische Interaktionen, BlueMap-Anbindung

## Regeln für jede Phase

1. Nach jeder Phase muss `./gradlew build` fehlerfrei durchlaufen.
2. Wichtige Änderungen im Dev-Client testen (`./gradlew runClient`) und kurz beschreiben, was getestet wurde.
3. Pro Phase ein eigener Commit. Nach jeder Phase anhalten und Leopold zeigen, was geändert wurde.
4. `MobAiService.java` (über 4000 Zeilen) nicht komplett umbauen. Gezielt ändern, neue Logik lieber in neue Klassen auslagern.
5. API-Keys niemals ins Repo. Nur über Config bzw. Einstellungs-Panel.

## Phase 0: Setup und Überblick

- Original unverändert bauen und starten, prüfen ob alles läuft.
- In `NOTES.md` festhalten: Wo werden LLM-Aufrufe gemacht (Server oder Client)? Wo läuft Spracherkennung, wo TTS? Wie funktioniert das bestehende Gossip-System (`gossipEnabled`, `gossipRadius`, `gossipChance`)? Was liegt in `minecraft_ai_mob_personas/`?
- Klären: Braucht bei Multiplayer jeder Spieler einen eigenen API-Key, oder reicht einer auf dem Server?

## Phase 1: Kostenlose KI über Groq

- Groq ist OpenAI-kompatibel und bietet Chat und Whisper über denselben Endpunkt. Die vorhandene `OpenAiClient`-Logik sollte daher fast unverändert funktionieren.
- Standardwerte in `MobMindConfig`:
  - `apiEndpoint = https://api.groq.com/openai/v1`
  - `chatModel = llama-3.3-70b-versatile`
  - `sttModel = whisper-large-v3-turbo`
  - `sttLanguage = de`
- Prüfen, ob Groq alle Parameter akzeptiert, die `OpenAiClient` mitschickt (z. B. `max_tokens`, `response_format`). Nicht unterstützte Felder entfernen oder abfangen.
- HTTP 429 (Rate Limit) sauber behandeln: kurz warten, dann `offlineFallback` nutzen und im HUD einen verständlichen Hinweis auf Deutsch zeigen.
- Neues Config-Feld `smalltalkModel` (Standard `llama-3.1-8b-instant`, höheres Tageslimit) für Phase 4 schon anlegen.
- Hinweis: Wenn kein lokales STT-Modell konfiguriert ist, nutzt `VoiceManager` automatisch die API-Transkription. Deutsche Spracherkennung sollte damit schon nach dieser Phase funktionieren.

## Phase 2: Deutsch statt Englisch

- Den Bool-Schalter `isEnglishUi()` durch ein Enum ersetzen, z. B. `UiLanguage { ZH, EN, DE }`, gesteuert über die Minecraft-Spracheinstellung (`de_de` -> DE).
- Die Helfer `t(zh, en)` um Deutsch erweitern. Wo möglich, Texte in Sprachdateien verschieben (`assets/mobmind/lang/de_de.json`) statt drei Strings im Code.
- Die harte Prompt-Regel "reply in ENGLISH ONLY" für DE ersetzen durch: Antworte ausschließlich auf Deutsch, locker und umgangssprachlich, egal in welcher Sprache der Spieler spricht.
- Personas: entweder übersetzen oder im Prompt festlegen, dass die Persona auf Deutsch gesprochen wird.
- `ItemCatalog`: deutsche Itemnamen aus Minecrafts `de_de.json` erzeugen (`items_de.json`), analog zu `items_zh.json`. Deutsche Zahlwörter ("drei", "zehn") parsen, analog zu `chineseNumber`.
- HUD-Texte (`MobMindHud`) und Fehlermeldungen übersetzen.

## Phase 3: Lokale Sprach-Engine (optional, für Offline-Betrieb)

- SenseVoice kann kein Deutsch. In `SherpaEngine` / `SherpaLocal` einen Whisper-Pfad über sherpa-onnx ergänzen (Encoder, Decoder, Tokens, `language=de`).
- Neues Config-Feld `sttEngine = sensevoice | whisper`.
- TTS: Prüfen, ob eine deutsche Piper-Stimme über den vorhandenen VITS-Pfad (`--vits-model`) läuft. Falls ja, im README dokumentieren, welche Dateien man herunterladen muss.
- Mehrere deutsche Stimmen für verschiedene Mob-Typen ermöglichen (bestehendes `ttsVoicePool` nutzen).

## Phase 4: Smalltalk zwischen Mobs

- Auf dem bestehenden Gossip-System aufbauen, nichts doppelt bauen.
- Nur auslösen, wenn ein Spieler in Hörweite ist (ca. 16 Blöcke). Ohne Zuhörer keine API-Aufrufe.
- Cooldown pro Mob-Paar und ein globales Budget (z. B. max. Smalltalk-Aufrufe pro Minute), beides konfigurierbar. Groq-Free-Tier-Limits beachten.
- Für Smalltalk `smalltalkModel` nutzen, für direkte Gespräche mit Spielern `chatModel`.
- Gespräche sollen sich auf echten Kontext beziehen: Wetter, Tageszeit, Raids, was Spieler in der Nähe gerade getan haben, Gerüchte aus dem Gossip-System.
- Ein Aufruf erzeugt am besten den ganzen Dialog (z. B. 2 bis 4 Zeilen, als JSON), statt pro Zeile einen eigenen Request.
- Anzeige als Chat oder Sprechblase, TTS mit Lautstärke nach Entfernung.

## Phase 5: Artspezifische Interaktionen

- Datengetrieben als JSON (`data/mobmind/relations/*.json`), damit Leopold selbst Beziehungen ergänzen kann.
- Pro Beziehung: Arten A und B, Ton (z. B. respektvoll, panisch, spöttisch, neidisch), Auslöser (Sichtkontakt, Angriff, Handel), optional leichtes Verhalten mit vorhandenen Goals (Flucht, Hinschauen, Folgen). Keine komplett neue Pathfinding-KI.
- Startbeispiele:
  - Villager und Eisengolem: Villager schwärmen vom Golem, Golem ist wortkarg
  - Villager und Zombie: Panik, Beleidigungen beim Weglaufen
  - Fahrender Händler und Villager: Konkurrenz, lästern über Preise
  - Pillager und Villager: Pillager verspotten Villager
  - Creeper und Katze: Creeper hat Panik
  - Skelett und Wolf: Skelett ist nervös
  - Piglin und Spieler mit Goldrüstung: übertrieben freundlich
  - Hexe und Villager: gegenseitiges Misstrauen

## Phase 6: BlueMap-Integration

- Zuerst prüfen, ob es BlueMap und die BlueMap-API für Minecraft 26.2 gibt. Falls nicht: Phase überspringen und notieren.
- BlueMap als optionale Abhängigkeit einbinden. Die Mod muss auch ohne BlueMap starten.
- Ideen:
  - Marker-Set "MobMind" mit benannten Mobs (Name, Stimmung, Beziehung zu Spielern)
  - Mobs kennen benannte BlueMap-Marker in der Nähe und erwähnen sie im Gespräch ("Die Fabrik von X da drüben ...")
  - Optional: Marker für Orte, an denen gerade viel getratscht wird

## Phase 7: Release

- README auf Deutsch: Installation, Groq-Key holen, Modelle für Offline-Betrieb, Credits ans Original.
- MIT-Lizenz unverändert lassen.
- Optional: Das Sprachsystem aus Phase 2 als eigenen Pull Request beim Original einreichen.
