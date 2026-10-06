# NOTES – MobMind-DE

Arbeitsnotizen zum Fork. Wird nach jeder Phase aus PLAN.md aktualisiert.

## Stand

| Phase | Status | Datum |
|---|---|---|
| 0 – Setup und Überblick | erledigt (Analyse). Build nur teilweise geprüft, weil in der Cloud-Umgebung JDK 25 fehlt (siehe unten) | 2026-10-06 |

---

## 1. Build-Umgebung

### Was das Projekt braucht

- **JDK 25** (`options.release = 25`, `sourceCompatibility = 25`, `fabric.mod.json` → `"java": ">=25"`)
- Gradle 9.5.0 (lädt der Wrapper selbst herunter, nichts zu installieren)
- Fabric Loom 1.17-SNAPSHOT (aufgelöst zu 1.17.21), Loader 0.19.3, Fabric API 0.152.2+26.2, Minecraft 26.2
- sherpa-onnx 1.13.4 kommt aus dem lokalen Repo `libs/repo` (nur `win-x64`-Natives)

### Ergebnis des Build-Tests (Cloud-Container, 2026-10-06)

- Installiert ist nur **OpenJDK 21**, kein JDK 25.
- `./gradlew` lässt sich unter Linux nicht direkt starten: Die Datei ist im Git ohne Ausführungsrecht eingecheckt (Modus `100644`). Unter Windows (`gradlew.bat`) ist das egal. Unter Linux/macOS hilft `sh gradlew build` oder einmalig `git update-index --chmod=+x gradlew` (noch nicht gemacht, weil Phase 0 nichts ändern soll).
- Maven Central hat Gradle aus dem Container heraus mit **HTTP 429** gedrosselt. Das liegt am Netz der Cloud-Umgebung, nicht am Projekt: Mit einem temporären Init-Skript, das nur in der Sandbox auf den Google-Spiegel von Maven Central umleitet (nicht im Repo), lief die Konfiguration komplett durch, inklusive Minecraft-Download und Loom-Setup.
- Danach bricht der Build beim Kompilieren ab:
  ```
  > Task :compileJava FAILED
  error: release version 25 not supported
  ```
  Das ist genau der fehlende JDK 25. Am Code liegt es nicht.
- Dev-Client (`runClient`) konnte deshalb nicht gestartet werden. In einem Container ohne Bildschirm ginge das ohnehin nur eingeschränkt.

### Was Leopold lokal installieren muss

1. **JDK 25** (LTS), z. B. *Eclipse Temurin 25* von https://adoptium.net/temurin/releases/?version=25
   - Windows: MSI-Installer, Haken bei „Set JAVA_HOME“ und „Add to PATH“ setzen.
   - Prüfen: `java -version` muss `25` zeigen. Wenn mehrere JDKs installiert sind: `JAVA_HOME` auf das JDK 25 setzen.
2. Sonst nichts: Gradle, Loom, Minecraft und Fabric API lädt der Wrapper beim ersten `gradlew build` selbst.
3. Danach: `gradlew build` (Windows) bzw. `sh gradlew build` (Linux/macOS), anschließend `gradlew runClient`.

---

## 2. Architektur-Überblick (Antworten auf die Fragen aus Phase 0)

### Wo laufen die LLM-Aufrufe? → **Auf dem Server**

- Alle Chat-Completions laufen über `MobAiService.respond()` (`src/main/java/com/mobmind/ai/MobAiService.java:2168`). Die Methode baut den Prompt (`buildMessages`, Zeile 2325), ruft asynchron `OpenAiClient.chat()` auf (`src/main/java/com/mobmind/ai/OpenAiClient.java:35`), parst das JSON und wendet in `finish()` (Zeile 2231) Freundschaft, Aktionen, Handel usw. auf dem Server-Thread an.
- Die Antwort geht als `ReplyPayload` an alle Clients, die den Mob tracken (`MobPackets.java`).
- Eingaben kommen auf zwei Wegen an:
  - Normaler Chat → `ServerMessageEvents.CHAT_MESSAGE` (`MobMindMod.java:530`) → `handleChatMessage`: Bis zu 3 Mobs in Hörweite antworten.
  - Sprache → Der Client transkribiert und schickt den Text per `SpeakPayload` → `handleSpeak` an genau einen Mob.
- `OpenAiClient.chat` hat zwei Wege:
  - Lokales Ollama (localhost) → natives `/api/chat`
  - Alles andere → OpenAI-kompatibles `/chat/completions` (`chatOpenAi`, Zeile 171). Groq landet hier.
- Gedrosselt wird nur grob: höchstens **3 gleichzeitige** API-Aufrufe (`MobMindExecutor.API_SLOTS`). Sind alle belegt, kommt die Meldung „api_busy“. Dazu je Mob 2 s Cooldown bei Spielereingaben und eigene Cooldowns pro Auslöser.
- Es gibt 74 `respond(...)`-Stellen in `MobAiService` (Begrüßung, Schaden, Gossip, Handel, Diebstahl, Feuer, Leine, Angeln, Reiten …). Jede davon ist ein LLM-Aufruf. Das ist wichtig für das Groq-Budget.
- Ohne API-Key oder bei einem Fehler: `offlineFallback` → `offlineReply()` mit eingebauten Sätzen.
- Die Sprache wird **pro Spieler** gewählt: Der Client schickt beim Join seinen Sprachcode (`LanguagePayload`), der Server speichert ihn in `MobMindState.PLAYER_LANGUAGE`. `isPlayerEnglish()` prüft nur `startsWith("en")`, alles andere gilt als Chinesisch. Dort setzt Phase 2 an.

### Wo läuft die Spracherkennung (STT)? → **Auf dem Client**

`VoiceManager.transcribe()` (`src/main/java/com/mobmind/client/VoiceManager.java:136`). Taste V (Umschalten) bzw. Strg+V (gedrückt halten). Reihenfolge:

1. **sherpa-onnx per JNI** (`SherpaLocal`), Modell bleibt im Speicher. Nur SenseVoice, also kein Deutsch. Native Bibliothek nur `win-x64`.
2. **sherpa-onnx als `.exe`-Prozess** (`SherpaEngine`), sucht `sherpa-onnx-offline.exe`, läuft also nur unter Windows.
3. **Cloud-API** `OpenAiClient.transcribe()` → `/audio/transcriptions`, mit dem **API-Key aus der Client-Config**.

Danach geht der Text per `SpeakPayload` an den Server. Ohne lokales Modell und ohne Key auf dem Client erscheint `stt_not_configured`.

### Wo läuft die Sprachausgabe (TTS)? → **Auf dem Client**

`MobMindClient.playVoice()` (`src/main/java/com/mobmind/client/MobMindClient.java:129`). Gleiche Reihenfolge: JNI (kokoro/vits) → `.exe` → Cloud-API `/audio/speech` (`ttsModel`, `ttsVoice`, `response_format: wav`, mit Client-Key).

- Abgespielt wird **nur beim Spieler, der das Gespräch ausgelöst hat** (Vergleich `speakerName`, Zeile 50). Andere Spieler in der Nähe sehen den Text, hören aber nichts. Für Smalltalk in Phase 4 muss das anders laufen (Zuhörer in Reichweite, Lautstärke nach Entfernung).
- Die Stimme (`voiceId`) wird pro Mob beim ersten Erzeugen aus `ttsVoicePool` gezogen und im Mob-State gespeichert.

### Wie funktioniert das Gossip-System?

Der Kern ist `spreadGossip()` (`MobAiService.java:448`). Aufgerufen wird es **nur** aus `onHurtByPlayer()`, also wenn ein Spieler einen unterstützten Mob schlägt oder tötet. Ein dauerhaftes Gerüchte-Gedächtnis gibt es nicht. Ablauf:

1. Abbruch, wenn `gossipEnabled = false`. Cooldown: **30 s pro Opfer**.
2. Gesucht werden Artgenossen (gleicher `EntityType`) im Radius `gossipRadius` (Standard **24** Blöcke). Die Liste wird gemischt, höchstens **3** werden berücksichtigt.
3. Sonderfall: Ist das Opfer ein Villager oder fahrender Händler, bekommen **alle Eisengolems** im Radius ohne Würfelwurf `-gossipPenalty` Freundschaft, werden 4 Minuten provoziert und greifen den Spieler an.
4. Für jeden der höchstens 3 Artgenossen wird mit `gossipChance` (Standard **60 %**) gewürfelt, ob er das Gerücht glaubt. Wenn ja:
   - Freundschaft `-gossipPenalty` (Standard **6**)
   - Feindliche oder neutrale Mobs werden 3 Minuten provoziert und greifen bei Sichtkontakt an.
   - Passive Mobs (Villager usw.) bekommen 1 Minute lang den Befehl FLEE.
   - Mit `gossipReactChance` (Standard **35 %**) gibt es einen **LLM-Aufruf**: Der Mob schimpft oder warnt seine Artgenossen (`respond(..., applyActions=false)`). Der Prompt unterscheidet zwischen „getötet“ und „angegriffen“.
5. Ein Schlag kann also bis zu 1 (Reaktion des Opfers, 20 s Cooldown) + 3 (Gossip) = **4 LLM-Aufrufe** auslösen.

Daneben gibt es verwandte Mechanismen, die keine Config-Schalter haben:

- **`tryVillagerGossip`** (`MobAiService.java:1446`), aufgerufen alle 30 s aus `MobMindMod`. Wenn mindestens 2 Villager im Umkreis von 16 Blöcken sind, der Spieler mehr als 4 Blöcke entfernt ist und ein 25-%-Wurf gelingt (60 s Cooldown pro Spieler), sieht der Spieler zwei graue Systemnachrichten mit **fest eingebauten Satzfetzen** (nach Freundschaft bzw. Grudges). **Kein LLM-Aufruf.**
- **`notifyNitwitGossip`**: Redet ein Spieler mit einem Nitwit, mischen sich bis zu 2 andere Villager ein (LLM, 30 s Cooldown).
- **`notifyCrowdOpinion`**: „Zeig allen mein Haus“ → bis zu 3 Mobs geben ihren Kommentar ab (LLM).
- **`tryRandomGreeting`**: alle 10 s, Mobs im Umkreis von 8 Blöcken mit Sichtkontakt, Freundschaft ≥ 25, 10 Minuten Cooldown pro Mob (LLM).
- Das Gossip-System von Vanilla-Villagern (`GossipContainer`) wird nicht angefasst.

Für Phase 4 heißt das: Gossip ist bisher **ereignisgetrieben**. Es reagiert auf Angriffe und hat kein eigenes Gedächtnis. Smalltalk sollte an `spreadGossip` und `tryVillagerGossip` andocken und zusätzlich die Ereignisse in einen kleinen Gerüchte-Puffer schreiben, auf den sich der Smalltalk-Prompt beziehen kann.

### Was liegt in `minecraft_ai_mob_personas/`?

- Das ist das **ursprüngliche chinesische Persona-Paket**: 50 `.txt`-Dateien mit chinesischen Dateinamen (z. B. `村民.txt` = Villager) und eine `README.md`.
- Es wird **weder vom Code noch vom Build** benutzt. Es ist Quellmaterial.
- Zur Laufzeit lädt die Mod die Kopien in `src/main/resources/assets/mobmind/personas/`:
  - `index.json`: Schlüssel → `entity` (z. B. `minecraft:villager`), `baby`, `goodPercent`, `goodLabel`, `evilLabel`
  - `<schlüssel>.txt` mit englischen Dateinamen, Inhalt chinesisch
  - 45 Dateien sind byte-identisch mit dem Quellpaket. Die anderen 5 (`copper_golem`, `parched`, `hoglin`, `witch`, `liufangguai` = Sulfur Cube) unterscheiden sich nur bei Anführungszeichen und Zeilenende. In `liufangguai.txt` wurde außerdem eine angehängte ChatGPT-Bemerkung entfernt.
- Aufbau jeder Persona: Grundidentität, Charakter-Wahrscheinlichkeit (gut/böse), Ausprägungen, Verhaltensregeln, Sprechstil, Auslöser für Freundschaft/Feindschaft, Rollenspiel-Anweisung.
- `PersonaRegistry` lädt `index.json`. **Nur Mobs mit Persona bekommen KI** (`supports()`), normale Tiere nicht. Gut/Böse wird deterministisch aus der Entity-UUID gewürfelt (`rollAlignment`) und bleibt danach fest.
- Name, Geselligkeit, Temperament, Humor und Stimme kommen aus `PersonalityGenerator` (chinesische Namen, englische über `generateEnglishName`). Für Deutsch fehlt beides noch (Phase 2).

---

## 3. Multiplayer: ein API-Key oder einer pro Spieler?

**Für Chat reicht ein Key auf dem Server.** Für Sprache braucht jeder Spieler selbst etwas.

| Funktion | Läuft auf | Wessen Config/Key |
|---|---|---|
| Mob-Antworten (LLM) | Server | `config/mobmind.json` **des Servers** |
| Spracheingabe (STT) | Client | Client-Config: lokales sherpa-Modell **oder eigener Key** |
| Sprachausgabe (TTS) | Client | Client-Config: lokales sherpa-Modell **oder eigener Key** |

- Auf einem Dedicated Server trägt man den Key von Hand in die `config/mobmind.json` auf dem Server ein. Das Einstellungsfenster (Strg+K / Mod Menu) ist rein clientseitig und ändert nur die **lokale** Config des Spielers. Auf einem fremden Server wirken also Gossip-Einstellungen, Chat-Modell usw. aus dem Fenster nicht. Das sollten wir in einer späteren Phase im Fenster kenntlich machen.
- Spieler ohne eigenen Key können trotzdem **per Chat tippen**, die Mobs antworten über den Server-Key. Nur Mikrofon und Sprachausgabe fallen weg.
- Lokales STT/TTS geht derzeit nur unter Windows (`win-x64`-Natives bzw. `.exe`). Linux- und Mac-Clients brauchen für Sprache also immer die Cloud-API.
- Der Key wird nie zwischen Client und Server übertragen. Gut so, und das soll auch so bleiben.
- Mit Groq gilt: Ein Server-Key heißt, **alle Spieler teilen sich das Free-Tier-Limit** dieses Keys (Requests/Tokens pro Minute und pro Tag). Deshalb sind in Phase 4 Budget und Cooldowns wichtig.
- Die Mod (`"environment": "*"`) gehört auf Server **und** Clients, weil Antworten, HUD und Sprache über eigene Netzwerkpakete laufen.

---

## 4. Create und BlueMap für Minecraft 26.2 (Fabric) – Stand 2026-10-06

### BlueMap → **ja, verfügbar**

- Offizielle Fabric-Builds für 26.2 gibt es seit **5.21 (16.06.2026)**. Aktuell ist **5.28-fabric (25.09.2026)** für *Fabric 26.1–26.3*. Benötigt Fabric API.
  - https://modrinth.com/plugin/bluemap
  - https://github.com/BlueMap-Minecraft/BlueMap/releases
- BlueMap 5.28 bringt **BlueMap API 2.8.1** mit.
- Für Phase 6: Die API einbinden mit
  ```gradle
  repositories { maven { url = uri("https://repo.bluecolored.de/releases") } }
  dependencies { compileOnly "de.bluecolored:bluemap-api:2.8.1" } // nicht shaden / nicht include
  ```
  und erst in `BlueMapAPI.onEnable(api -> …)` zugreifen. Dann startet die Mod auch ohne BlueMap. Zusätzlich in `fabric.mod.json` unter `suggests` eintragen.

### Create → **nur inoffiziell**

- Das **offizielle Create** (simibubi) gibt es nur für Forge/NeoForge bis **1.21.1**. Für 26.x existiert keine Version.
- Der **offizielle Fabric-Port „Create Fabric“** endet bei **1.20.1**.
- Der **inoffizielle Fabric-Port „Create Fly“** (ZurrTum, CC0) hat die Version `26.2-rc-2-6.0.9-1` („Create 6.0.9 build-1 for 26.2-rc-2“, 14.06.2026). Sie ist auf Modrinth für `26.2-rc-2` **und `26.2`** markiert. Seit dem 15.06.2026 gab es keinen neueren Build.
  - https://modrinth.com/mod/create-fly
  - https://github.com/ZurrTum/Create-Fly
  - Einschränkungen laut README: nicht offiziell (Bugs nicht an das Create-Team melden), Kompatibilität mit Fabric-Events und anderen Mods noch unvollständig, alte Welten können Daten verlieren, mit Shadern ist die Flywheel-Optimierung aus.

### Fazit für den Server

- **MobMind-DE + BlueMap auf 26.2 Fabric: machbar.**
- **Create auf 26.2 Fabric: nur über „Create Fly“** (inoffiziell, gebaut gegen 26.2-rc-2). Vor dem Einsatz auf dem echten Server unbedingt mit einer Kopie der Welt testen, und zwar zusammen mit MobMind, BlueMap und Fabric API. Wenn Create Pflicht ist und Stabilität wichtig ist, bleibt sonst nur NeoForge 1.21.1. Darauf läuft diese Mod (Fabric, 26.2) aber nicht.

---

## 5. Auffälligkeiten und Hinweise für spätere Phasen

- **Phase 1 (Groq):** `chatOpenAi` schickt immer `"think": false` und `"stream": false` mit (`OpenAiClient.java:179`). Prüfen, ob Groq unbekannte Felder wie `think` ablehnt (HTTP 400). Falls ja, nur bei Ollama oder lokalen Endpunkten senden. `max_tokens` = 2048 ist für 60-Zeichen-Antworten unnötig hoch und kostet bei Groq Token-Budget.
- **Phase 1:** HTTP-Fehler werden nur als `IOException("... HTTP 429 ...")` weitergereicht. Für die 429-Behandlung muss der Statuscode sauber herausgereicht werden (z. B. eigene Exception-Klasse).
- **Phase 1/3 (TTS):** Groq hat meines Wissens keine deutsche TTS-Stimme. `speak()` ist fest auf das OpenAI-Format eingestellt (`tts-1`/`alloy`). Deutsche Sprachausgabe läuft also über lokales Piper/VITS (Phase 3) oder einen anderen Anbieter.
- **Phase 2:** Fest eingebaute Sprachregeln stehen im System-Prompt (`MobAiService.java:2489` und das chinesische Gegenstück). `t(zh, en)` gibt es zweimal: mit und ohne Spieler-UUID. Die Variante ohne UUID errät die Sprache über die Server-Locale.
- `TalkScreen` (Texteingabe mit Taste Y) ist **toter Code**, er wird nirgends geöffnet.
- Der Mob-Zustand wird pro Welt in `<welt>/mobmind.json` gespeichert. Gleicher Dateiname wie die Config, durch die `.gitignore` ebenfalls abgedeckt.
- Fabric API im Projekt: `0.152.2+26.2`. Neuester 26.2-Build auf Modrinth ist `0.161.0+26.2` (18.09.2026). Kein Handlungsbedarf, nur zur Info: Auf dem Server reicht jede Fabric API ≥ 0.152.2 für 26.2.
- `MobAiService.java` hat 4321 Zeilen. Wie vereinbart nur gezielt ändern, neue Logik in eigene Klassen (z. B. `ai/GroqRateLimiter`, `ai/SmalltalkService`, `relations/…`).

## 6. Sicherheit

- `.gitignore` ergänzt: `/config/`, `mobmind.json` (überall), `.env`, `.env.*`, `*.secret`. Die Laufzeit-Config liegt ohnehin unter `run/`, das war schon ignoriert.
- Mit `git check-ignore` geprüft: `run/config/mobmind.json`, `config/mobmind.json` und `.env` werden ignoriert. `src/.../config/MobMindConfig.java`, `mobmind.mixins.json` und `fabric.mod.json` werden weiter versioniert.
- In der Git-Historie wurde nach typischen Key-Mustern gesucht (`sk-…`, `gsk_…`, `ghp_…`, `"apiKey": "…"`): **nichts gefunden**.
