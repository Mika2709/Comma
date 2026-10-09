# 13 Handy und Voice

Ziel: Wichtige Meldungen erreichen das gesperrte iPhone zuverlässig, mit Text, ohne Doppelungen (E56). Auf dem iPhone gibt es denselben Chat, das Board, Reaktionen, Picker und Freigaben. Voice-to-Voice in der App, keine Telefonanrufe (E88).

Gewichtung: Puzzleteil (E8). Die iOS-App existiert schon im Repo, APNs-Code auch.

Pfad-Abkürzungen: `sa/` = `systems/apps/`, `cl/` = `clients/`.

---

## Ist-Zustand (Stand `ba4f55d`)

- iOS-, Watch- und Live-Activity-Code in `cl/apps/apple`. APNs auf dem Server (`sa/comma_core/lib/comma/notifications.ex`, `notifications/apns.ex`), Konfiguration `comma.apns` (`systems/config/config.example.json:196`, `sa/salix_store/lib/salix_store/config_json.ex:145,242-306`).
- Push nur für Task-Status (`notifications.ex:9-11`, `@attention ~w(ready_for_review escalated)`), Listener `sa/comma_web/lib/comma_web/native_push_listener.ex` startet nur mit APNs-Konfiguration (`:12-15`). Keine Push für Chat- oder proaktive Nachrichten.
- Bundle-IDs in `apns.ex:13` hart: `surf.comma.ios`, `surf.comma.ios.dev`.
- iOS registriert APNs beim Start (`cl/apps/apple/iOS/PhoneAppDelegate.swift:25,87-90`), Benachrichtigungen sind opt-in (`docs/clients.md:94`).
- `app_active` zählt nur Electron-Sessions der letzten 10 Minuten (`sa/comma_core/lib/comma/accounts/sessions.ex:208-220`, gesetzt über `POST /v1/comma/auth/session/activity`, `sa/comma_web/lib/comma_web/router.ex:590-599`).
- Ausweichweg heute: Der Router sendet selbst an Telegram oder WeChat, wenn die Desktop-App inaktiv ist (`sa/comma_web/lib/comma_web/proactive_delivery.ex:31-56`, `resources/salix-system-files/skills/proactive/SKILL.md:36-48`). Der Server sendet nie selbst.
- Telegram-Setup: `systems/config/runtime.exs:651-720` (`COMMA_TELEGRAM_ENABLED`, `_BOT_TOKEN`, `_BOT_USERNAME`, `_PUBLIC_BASE_URL`, `_WEBHOOK_SECRET`), Webhook-Registrierung per `bin/comma eval "CommaWeb.TelegramWebhook.apply(dry_run: false)"`. Nirgends dokumentiert, nicht in `selfhost/configure.py`.
- Voice: Server vorhanden (`docs/messaging-voice.md:80-200`): `SalixVoice.CallActor`, GPT-Live (`gpt_live_model` `gpt-live-1`, `wss://api.openai.com/v1/live/sessions`), WebSocket-API `GET /v1/agent-groups/{group_id}/voice` mit Subprotokoll `comma.voice.v1`, Header `Bearer salix_vk_…`, Formate `pcmu_8k`/`pcm16_24k`. Browser können den Header nicht setzen (`:165`). Einstellungen über `/dash/voice` (`sa/salix_voice/lib/salix_voice/settings.ex:17-31`), `enabled` standardmäßig aus, braucht `openai_api_key`. Voice-Stream-Token braucht `SalixStore.Crypto.derived_key` (`salix_voice/stream_token.ex:82`), das in Selbsthost fehlt (`02`).
- Der Router bekommt `voice.call_started`, Delegationen mit Transkript und `voice.call_ended` (`docs/messaging-voice.md:84-87`). Das Transkript im `CallActor` ist nicht dauerhaft.
- Keine Live-Voice-Oberfläche in Electron oder iOS. Der Mikrofon-Knopf im Composer nimmt nur eine Audiodatei auf (`cl/packages/app/src/components/chat/composer/Composer.tsx:164-320`).

---

## 1. iPhone-App bauen und installieren

Der Nutzer braucht dafür keinen Mac: Gebaut wird in GitHub Actions auf einem macOS-Runner.

1. Apple Developer Program (99 $ pro Jahr). Im Developer-Portal: eigene Bundle-ID (z.B. `de.<nutzer>.comma`), App-Group, Push-Capability. APNs-Key (`.p8`, läuft nicht ab), Key-ID, Team-ID. App-Store-Connect-API-Key für den Upload.
2. `apns.ex:13`: Bundle-IDs aus der Konfiguration lesen (`comma.apns.bundle_id`) statt hart.
3. Neuer Workflow `.github/workflows/ios-testflight.yml` im Fork: baut die App mit `COMMA_PHONE_BUNDLE_ID`, `COMMA_API_BASE_URL` (die eigene Domain, `02`) und `COMMA_PUSH_ENVIRONMENT=production` (siehe `cl/apps/apple/README.md`), signiert automatisch über den App-Store-Connect-API-Key (`-allowProvisioningUpdates`), lädt zu TestFlight hoch. Auslöser: Änderungen unter `clients/apps/apple/` oder `clients/packages/chat-contract/`, plus Zeitplan alle 60 Tage (TestFlight-Builds laufen nach 90 Tagen ab).
4. Secrets im GitHub-Repo: ASC-Key, Key-ID, Issuer-ID, Team-ID. Nie im Code.
5. TestFlight-Builds bekommen Production-Push-Tokens. Server-Konfiguration `comma.apns.production` mit dem `.p8`-Key.
6. Der Nutzer installiert TestFlight und die App, meldet sich an, erlaubt Benachrichtigungen.

---

## 2. Push für Chat und Meldungen (F27, V67)

### Zustellregel (Server, deterministisch)
Neues Modul `Comma.HomeDelivery`, ausgelöst, wenn der Assistent eine Nachricht in Home anhängt (Abo auf die Conversation-Mutationen, Muster `native_push_listener.ex:29-35`):
1. Ist der Nutzer gerade an einem Gerät (Regel unter "Anwesenheit": letzte Eingabe unter 2 Minuten): nichts weiter. Der Nutzer sieht es im Chat.
2. Sonst: APNs-Push an alle registrierten iPhones des Nutzers.
3. Schlägt APNs fehl (kein Gerät registriert, `BadDeviceToken`, Timeout): Server sendet selbst an Telegram (Bot-API, an den Chat des Nutzers), falls eingerichtet.
4. Ein Kanal pro Nachricht, kein Rundversand (V68). Mehrere Nachrichten desselben Turns werden zu einer Push-Meldung zusammengefasst (erste Nachricht als Text, "+2 weitere").
5. Zustellstatus pro Nachricht: `gesendet`, `zugestellt` (APNs 200), `fehlgeschlagen`, `ausgewichen` (Telegram), gespeichert als Betriebsprojektion. Kein Wiederholen außer dem einen Ausweichschritt (V69).

Folge: Der Router muss nicht mehr selbst an Telegram senden. `proactive_delivery.ex` und der Abschnitt in `SKILL.md:36-48` werden vereinfacht: "Schreib in Home. Die Zustellung aufs Handy erledigt der Server."

### Inhalt
- Push mit Text der Nachricht (höchstens 180 Zeichen, Rest abgeschnitten mit "…"). Eigener Server, eigenes Handy: Text im Push ist ausdrücklich gewollt (Datenschutzentscheidung des Nutzers, im PR festhalten).
- Kategorien:
  - `nachricht`: Tippen öffnet den Chat.
  - `freigabe`: Aktionsknöpfe "Erlauben" und "Ablehnen" direkt in der Mitteilung. Sie rufen die Entscheidungsroute (`10`) mit dem Token aus dem Schlüsselbund der App. "Immer erlauben" nur in der App.
  - `frage`: Tippen öffnet den Picker (`09`).
- Reaktionen des Assistenten lösen keine Push aus.
- Task-Status-Push (bestehend) wird abgeschaltet, weil Helfer unsichtbar sind (`08`).

### Anwesenheit (F22)
Datei `systems/apps/comma_core/lib/comma/accounts/sessions.ex`:
- `touch_active/2` (`:183-206`) nimmt heute nur `client_kind` `electron` an und gibt sonst `{:error, :not_desktop_app}`. Neu: auch `ios` und `web` (nicht eingeschränkte Sessions, gleiche Bedingungen wie bei `electron`). Route bleibt `/v1/comma/auth/session/activity`.
- Der Web-Client meldet Aktivität wie Electron (`clients/apps/electron/src/main/modules/session/activity-reporter.ts`): alle 60 Sekunden, solange es in den letzten 5 Minuten eine Eingabe gab und der Tab sichtbar ist. Die iOS-App meldet im Vordergrund alle 60 Sekunden.
- Jeder Bericht trägt neu `idle_seconds`: Electron `powerMonitor.getSystemIdleTime()`, Web Sekunden seit dem letzten Eingabe-Ereignis im Tab, iOS 0 im Vordergrund. Der Server speichert daraus `last_input_at` = Empfangszeit minus `idle_seconds` an der Session. Grund: Ein Bericht allein heißt nur "Eingabe in den letzten 5 Minuten", das ist für die Push-Entscheidung zu grob.
- `HomeDelivery` nutzt eine eigene Abfrage `active_within?(user_id, 120)`: irgendeine Session (`electron`, `ios`, `web`) mit Bericht in den letzten 90 Sekunden und `last_input_at` in den letzten 120 Sekunden. `present?/2` (`:209-220`, 10 Minuten, nur `electron`) bleibt für die bestehenden Aufrufer unverändert.
- Tests: `touch_active` für `ios` und `web` ok, für eingeschränkte Sessions abgelehnt; kein Push bei einem Web-Bericht vor 30 Sekunden mit `idle_seconds` 60; Push bei einem Bericht vor 30 Sekunden mit `idle_seconds` 200; Push, wenn der letzte Bericht 3 Minuten alt ist.

---

## 3. iPhone-App: Funktionen

Ergänzungen in `cl/apps/apple` (SwiftUI), jeweils nativ:
- Board statt Bucket-Seitenleiste (`09` Abschnitt 1.8).
- Reaktionen anzeigen und setzen (`06`): Badge an der Blase, Long-Press öffnet Emoji-Auswahl.
- Rückfragen mit Picker (`09` Abschnitt 2): SwiftUI `DatePicker`, `Picker`, `Form`.
- Freigabekarten mit Details und Knöpfen (`10`).
- Dynamic-UI-Widgets über `WKWebView` (`09` Abschnitt 3).
- Deutsch (`12`).
- Voice-Modus (Abschnitt 5).
- Keine Task-Karten im Chat (`08`).

---

## 3b. Notlösung ohne Apple-Konto

Ohne Apple-Developer-Konto: den bestehenden Web-Client in Safari öffnen und "Zum Home-Bildschirm" hinzufügen (E80). Chat, Board und Freigaben funktionieren dort. Push kommt dann nur über Telegram. Widgets brauchen die Web-Freigabe aus `09` Abschnitt 3.

## 4. Telegram als Ausweichweg

- `selfhost/configure.py` übernimmt `COMMA_TELEGRAM_*` aus `.env` und schreibt `comma.telegram`.
- Nach dem Start einmal `bin/comma eval "CommaWeb.TelegramWebhook.apply(dry_run: false)"` (als Make-Ziel `make selfhost-telegram-webhook`).
- Kurzes Runbook: Bot bei @BotFather anlegen, Token, Username, Webhook-Secret (32 bis 256 Zeichen `[A-Za-z0-9_-]`), öffentliche HTTPS-URL.
- Der Nutzer verbindet seinen Telegram-Chat einmal mit Comma (bestehender Verbindungsweg). Schreibt er dort, landet es in derselben Router-Session.

---

## 5. Voice-to-Voice

### Server
- Voice einschalten über `/dash/voice`: `enabled: true`, `openai_api_key`, `gpt_live_model` bleibt `gpt-live-1`. Kein Twilio.
- Voice-API-Key für die Group anlegen (Route `/v1/comma/workspaces/:id/voice-api-keys`). Den Key bekommen nur die nativen Clients (Electron-Main, iOS-Schlüsselbund), nie der Renderer.
- Voraussetzung `derived_key` über den neuen Secret-Weg (`02` Selbsthost-Fix 3).
- Antwortlänge im Gespräch über das bestehende Voice-Profil (`RouterDecision`, jetzt über den Entscheidungs-Router, `14`).
- **Transkript in den Chat:** Bei `voice.call_ended` hängt der Server eine Zusammenfassung plus das volle Transkript als `app_event` `voice.call_summary` an Home an (über `ConversationServer`). Die Clients zeigen eine kompakte Blase "Sprachgespräch, 21:40, 3 min", aufklappbar. So erfasst das Gedächtnis auch Gesprochenes (`04`), und der Router weiß später davon.

### Electron
- Knopf "Sprechen" im Composer (neben dem bestehenden Aufnahme-Knopf).
- Native Fähigkeit über den Capability-Kernel (AGENTS.md: keine rohe IPC): neue Leaves in `cl/packages/native-bridge/src/capability-leaves.ts`: `voice.session.start` und `voice.session.stop` (Commands), `voice.session.state` (State), Audio-Frames als Events in beide Richtungen. Danach generieren und `pnpm --dir clients check:foundation`.
- Electron-Main öffnet die WebSocket-Verbindung mit `Authorization: Bearer salix_vk_…` und Subprotokoll `comma.voice.v1` (der Renderer kann den Header nicht setzen). Der Key liegt im SecureStore.
- Renderer: Mikrofon per `getUserMedia`, PCM16 24 kHz, Wiedergabe über Web Audio. Anzeige: Pegel, "hört zu" / "spricht", Auflegen.

### iPhone
- `AVAudioEngine` für Aufnahme und Wiedergabe (PCM16 24 kHz), `URLSessionWebSocketTask` mit Header und Subprotokoll. Knopf im Chat, Vollbild-Ansicht während des Gesprächs, läuft mit gesperrtem Bildschirm weiter (Audio-Background-Mode).

---

## 6. Abnahme

- Tests: `HomeDelivery` wählt genau einen Kanal; aktive Session unterdrückt Push; APNs-Fehler fällt auf Telegram; Zusammenfassung mehrerer Nachrichten; Freigabe-Aktion aus der Mitteilung; `voice.call_summary` landet in Home.
- Manuell: Abnahmetest 3 aus `16` (gesperrtes iPhone bekommt die wichtige Mail als Push), ein Sprachgespräch über Electron und eins über das iPhone, danach weiß der Assistent im Chat, was besprochen wurde.
