# 02 Infrastruktur, Anbieter und Selbsthosten

Ziel: Comma läuft dauerhaft auf einem eigenen Linux-VPS, unabhängig vom Laptop (E75, E76). Der Assistent hat einen eigenen Windows-Rechner. Alles ist über ein privates Netz verbunden. Die bekannten Selbsthost-Brüche sind repariert. Backups laufen automatisch und werden geprüft.

Kleinkram hier wird nicht mit dem Nutzer diskutiert, sondern erledigt (E81).

Pfad-Abkürzungen: `sa/` = `systems/apps/`.

---

## 1. Maschinen und Netz

| Maschine | Zweck | Wahl |
|---|---|---|
| Linux-VPS | Comma-Server (Docker Compose), Gedächtnisdienst, Reverse Proxy | Hetzner Cloud CPX42 (8 vCPU, 16 GB RAM), Ubuntu 24.04, Standort Deutschland |
| Windows-VPS | Rechner des Assistenten (Desktop, Chrome) | Contabo Cloud VPS 30 mit Windows Server 2022, Details in `11` |
| Rechner des Nutzers | Lokale Steuerung bei Bedarf | macOS oder Windows (`11` Abschnitt 5) |
| iPhone | App, Push, RDP-Übernahme | `13` |

Netz: Tailscale (Personal, kostenlos).
- Tags: `tag:comma-server` (Linux-VPS), `tag:agent-pc` (Windows-VPS). Geräte des Nutzers als Nutzergeräte.
- ACL: `tag:comma-server` darf `tag:agent-pc` auf 9222 (CDP) und 8765 (Windows-MCP). Nutzergeräte dürfen `tag:agent-pc` auf 3389 (RDP) und den Rechner des Nutzers. Sonst nichts.
- Die Docker-Container erreichen Tailnet-Adressen über das Host-Routing (Tailscale läuft auf dem Host).

Öffentlich erreichbar ist nur der Linux-VPS auf 80/443 (Reverse Proxy). Composio- und Telegram-Webhooks brauchen öffentliches HTTPS. Firewall (UFW): eingehend nur 80, 443, SSH nur über Tailscale.

---

## 2. Domain und HTTPS

Comma braucht vier Origins unter einer registrierbaren Domain (`systems/DEPLOYMENT.md`, Abschnitt "Public access"). Dazu kommt ein fünfter Hostname für Web-Widgets (letzte Zeile):

| Variable | Beispiel | Proxy-Ziel |
|---|---|---|
| `COMMA_PUBLIC_URL` | `https://app.<domain>` | 127.0.0.1:8080 |
| `COMMA_ADMIN_URL` | `https://admin.<domain>` | 127.0.0.1:8082 |
| `COMMA_API_URL` | `https://api.<domain>` | 127.0.0.1:8081 |
| `COMMA_SALIX_URL` | `https://salix.<domain>` | 127.0.0.1:4000 |
| `COMMA_UI_SANDBOX_URL` (neu) | `https://ui.<domain>` | 127.0.0.1:8083 (statische Datei im `web`-Container) |

Der fünfte Origin ist neu und nur für Web-Widgets (`09` Abschnitt 3, "iPhone und Web"):
- Der Web-Build schreibt `dynamicUiDocument()` (`clients/packages/chat-contract/src/dynamic-ui/runtime.ts`) als statische Datei `runtime/index.html` in ein eigenes Ausgabeverzeichnis. `selfhost/Web.Dockerfile` kopiert es nach `/srv/ui` und gibt Port 8083 frei. `selfhost/Caddyfile` (der Caddy im `web`-Container) bekommt einen Block `:8083`, der nur `/runtime/` aus `/srv/ui` liefert, jeder andere Pfad gibt 404, und dort die Header unten setzt. `compose.yaml` bindet 8083 wie 8080 bis 8082 an `127.0.0.1`. Der Caddy auf dem Host leitet `ui.<domain>` an `127.0.0.1:8083`.
- Header auf diesem Origin: `Content-Security-Policy` nach `09` Abschnitt 3 (Web-Variante von `dynamicUiCsp`) plus `frame-ancestors https://app.<domain>`, `Cache-Control: no-store`, `Referrer-Policy: no-referrer`. Caddy entfernt jedes `Set-Cookie`.
- Der Web-Client bekommt die URL über `COMMA_UI_SANDBOX_URL` (Build- bzw. Laufzeit-Config wie die anderen URLs).

- Reverse Proxy: Caddy auf dem Host (nicht im Compose), automatische Let's-Encrypt-Zertifikate, WebSocket und Streaming ohne Puffern (`flush_interval -1` für Event-Streams), Host- und Origin-Header durchreichen.
- DNS: fünf A-Records auf die öffentliche IPv4 des Linux-VPS.
- SMTP für Login-Codes ist bei öffentlichem Betrieb Pflicht (`selfhost/configure.py:38-40`). Zugang vom Nutzer (siehe Startfragen).
- `COMMA_OWNER_EMAIL` = Mail des Nutzers. `COMMA_ALLOW_SIGNUP` bleibt aus.

---

## 3. Anbieter und Modelle

| Zweck | Modell | Anbieter, ID, Weg |
|---|---|---|
| Hauptmodell (Router und Worker) | Nach Startfrage 1. Empfehlung Claude Opus 5.5 | Anthropic-API `claude-opus-5-5`, oder vorhandenes Claude-/Codex-Abo. Modell-Template mit `context_tokens` = echtes Fenster des Modells laut Anbieter, `cache_ttl_mode`/`cache_ttl_seconds` (`03` Abschnitt 5). Bei Anthropic zusätzlich Thinking-Bindung (`03` Abschnitt 7) |
| Cleanup-Crew, Erfasser, Auditor, Verifier 1 | DeepSeek V4.1 Flash | Together `deepseek-ai/DeepSeek-V4.1-Flash` (0,30 $ Ein, 1,20 $ Aus pro 1M), Chat-Completions. Ausweichweg OpenRouter `deepseek/deepseek-v4.1-flash`. Kontext laut Modellkarte bis 1M. Beim Einrichten das Together-Fenster mit einem echten Call prüfen: Ist es kleiner als 400k, läuft die Crew über den OpenRouter-Ausweichweg. Die Aufteilung nach Typ aus `03` Abschnitt 3 bleibt der letzte Ausweg. |
| Verifier 2 | Claude Haiku 5.5 | OpenRouter `anthropic/claude-haiku-5.5` (datierte Fassung pinnen, falls gelistet). Ausweichweg bei fehlgeschlagenem Prüf-Call: `openai/gpt-5.5-mini` über OpenRouter. Regel in `03` Abschnitt 4. |
| Entscheidungsmodelle | Jev 1.13, Luna, Clef, Clef-flash, Liquid d1 | OpenRouter Decisions (`14`) |
| Embedding | Qwen3-Embedding-8B, 1536 Dimensionen | DeepInfra (`04` Abschnitt 10) |
| Reranker | Qwen3-Reranker-4B (8B bei guter Latenz) | DeepInfra (`04` Abschnitt 10) |
| Voice | GPT-Live `gpt-live-1` | OpenAI (`13`) |
| Websuche | Exa | `search.exa_api_key` (bestehend) |

- Alle Nicht-Haupt-Modelle werden als Modell-Templates (Chat-Completions an die jeweilige Basis-URL) bzw. Config-Einträge angelegt, nicht hart im Code.
- Jede Modell-ID wird beim Einrichten mit einem echten Call geprüft und gepinnt (datierte Version, wo vorhanden). Schlägt der Call fehl, gilt die Ausweichregel aus `14` Abschnitt 5 (nächstes Modell derselben Zeile, Dashboard-Vermerk, Aktions-Gate nie ohne Gate). Für Crew, Erfasser, Auditor und Verifier 1 ist der Ausweichweg OpenRouter `deepseek/deepseek-v4.1-flash`.
- Prompt-Caching-Fakten (Stand 2026-10-08): Anthropic 5 min Standard oder 1 h (`ttl: "1h"`), Schreiben 1,25x bzw. 2x, Lesen 0,1x (Opus 5.5 0,05x). Lebensdauer zählt ab Start der Anfrage. Keep-alive mit `max_tokens: 0` offiziell unterstützt, aber nicht mit Streaming und nicht mit `thinking.type: enabled`. OpenAI: `prompt_cache_retention` `in_memory` (5 bis 10 min, höchstens 1 h) oder `24h`; ab GPT-5.6 stattdessen `prompt_cache_options.ttl = "30m"`. DeepSeek: Disk-Cache, Stunden bis Tage, nicht garantiert. Die gewählte Einstellung, `cache_ttl_seconds` und Keep-alive pro Anbieter stehen in der Tabelle in `03` Abschnitt 5.

---

## 4. Selbsthost-Reparaturen (V75)

1. **Build bricht ab:** `systems/Dockerfile:69` kopiert `website/package.json`, den Ordner gibt es nicht. Die Stage `bft-dashboard-builder` (`:64-73`) scheitert, und weil `elixir-builder` sie bei `:166` per `COPY --from` zieht, bricht `docker compose up --build`. Zeile 69 entfernen. Die Root-Skripte `dev/build/deploy:website` in `package.json:14-20` und der Eintrag `- website` in `pnpm-workspace.yaml:3` sind tot; entfernen (der Web-Build bricht daran nicht, aber es ist Altlast).
2. **Connector-Installation:** Das `runtime`-Image hat kein `/opt/comma/install-artifacts/release-descriptor.json` (`systems/Dockerfile:231-233` nur in der Stage `release`), der lokale Artefaktpfad gilt nur für dev (`systems/config/runtime.exs:3-8`), Ergebnis `connector_release_unavailable` (`sa/salix_env/lib/salix_env/device_install.ex:100-116`). `DEPLOYMENT.md:110` behauptet das Gegenteil.
   - Neue Dockerfile-Stage `connector-builder` (Go-Image) baut `salix-connect` für `darwin-arm64` und `darwin-amd64` aus `systems/connector/salix-connect`, erzeugt `release-descriptor.json` im erwarteten Format (`sa/salix_env/lib/salix_env/server_release_descriptor.ex`) und das `runtime`-Image kopiert beides nach `/opt/comma/install-artifacts`.
   - Der macOS-Swift-Helfer für Computer Use kann nicht unter Linux gebaut werden. Er wird im macOS-Workflow (`13` Abschnitt 1) mitgebaut und als Artefakt bereitgestellt; nur nötig, wenn der Rechner des Nutzers ein Mac ist.
3. **Schlüssel-Ableitung ohne Compute-URL:** `compute_workload_credential_secret` wird nur gesetzt, wenn `SALIX_COMPUTE_RUNTIME_BASE_URL` gesetzt ist (`systems/config/runtime.exs:21-22,31-41`). Ohne liefert `SalixStore.Crypto.derived_key/1` `{:error, :credential_sealer_unavailable}` (`sa/salix_store/lib/salix_store/crypto.ex:148-156`). Betroffen: Browser-Settings, Browser-Storage, Voice-Stream-Token, Slack-Konfiguration. Neue Umgebungsvariable `SALIX_CREDENTIAL_SECRET` (mindestens 32 Byte), die das Secret unabhängig setzt. `configure.py` erzeugt sie einmalig in `secrets.json`.
4. **Fehlende Konfiguration in `selfhost/configure.py`** (heute nur ENV aus `.env`, keine Prompts): ergänzen um
   - `decide` mit Anbietern und Einsätzen (`14`), Keys für OpenRouter und Together,
   - Gedächtnisdienst (URL, API-Key aus `secrets.json`, Bank-Präfix),
   - DeepInfra-Key (wird an den `memory`-Dienst gereicht),
   - `comma.apns` (Team-ID, Key-ID, `.p8`-Inhalt, Bundle-ID, Umgebung `production`),
   - `comma.telegram` (`13`),
   - Composio (Tenant-Settings mit `webhook_configured`, `sa/salix_web/lib/salix/control/composio_settings.ex:212,239`),
   - `SALIX_CREDENTIAL_SECRET` (Punkt 3),
   - Standard-Locale `de` und Standard-Zeitzone (`12`),
   - Rebuild- und Crew-Konfiguration (`03`),
   - Browser-Provider `local_cdp` mit URL (`11`).
   Voice bleibt in `/dash/voice` (liegt in `ctl`, nicht in `config.json`).
5. **Postgres mit pgvector:** Image `postgres:16-bookworm` durch `pgvector/pgvector:pg16` ersetzen (gleiches Datenformat, Volume bleibt). Gebraucht vom Gedächtnisdienst und von der semantischen Verlaufssuche (`04` Abschnitt 9). `configure` legt die DB `hindsight` und ihre Rolle an.
6. **Gedächtnisdienst in Compose:** neuer Dienst `memory`, Build aus `services/hindsight` (Target `api-only`, ohne lokale Modelle), nur im internen Netz, Gesundheitsprüfung `/health/ready`, Abhängigkeit von `postgres` und `migrate`. Umgebungsvariablen aus `04` Abschnitt 8 und 10.
7. **Doku-Drift** in `systems/DEPLOYMENT.md` korrigieren: Connector-Installation (nach Fix 2 stimmt sie), RAM (nach Messung eintragen), Telegram, `decide`, APNs, Voice, Gedächtnisdienst, Backup-Abschnitt (unten). `docs/messaging-voice.md:99` ("No voice prices are seeded") ist veraltet (Preise sind in `billing_core/priv/repo/migrations/20260928000001_seed_missing_staging_prices.exs:138-150` geseedet). `docs/identity-security.md:280` (`07`). Danach `pnpm docs:check`.
8. **Messung:** Nach dem ersten Start RAM und Platte unter Last messen und in `DEPLOYMENT.md` eintragen (heute "not benchmarked").

---

## 5. Backups und Wiederherstellung (V76)

`DEPLOYMENT.md` verlangt konsistente Sicherungen aller fünf Volumes (config, postgres, objects, clickhouse, redis) im gestoppten Zustand, zusammen mit dem Quellstand. Danach richtet sich der Plan.

- **Feste Reihenfolge der Nachtjobs** (Ortszeit des Nutzers, alle als Oban-Cron außer dem Backup):
  - 02:00 Evals (`15`),
  - 02:30 Richter-Lauf, danach Kalibrierung (`14` Abschnitte 3 und 4),
  - 03:00 Gedächtnispflege (`04`, Status-Sweep und Widerspruchs-Job),
  - 04:30 Backup (systemd-Timer).
- **Backup nächtlich 04:30** auf dem Linux-VPS, systemd-Timer mit `OnCalendar=*-*-* 04:30:00 Europe/Berlin` (Zeitzone des Nutzers aus Startfrage 18; der VPS selbst läuft auf UTC). Die Oban-Crons laufen mit derselben Zeitzone (`timezone` in der Oban-Cron-Konfiguration):
  0. Warten, bis kein Oban-Job der Queues für Evals, Richter, Kalibrierung und Gedächtnispflege mehr `executing` ist, höchstens 30 Minuten. Läuft danach noch einer, verschiebt sich das Backup um 30 Minuten (höchstens zweimal), dann läuft es trotzdem; der Job wird beim Stopp abgebrochen und von Oban neu gestartet.
  1. `docker compose stop` (nicht `down`).
  2. restic-Backup der Volume-Verzeichnisse plus `.env`, plus aktueller Git-Commit als Tag.
  3. `docker compose start`. Ziel: unter 5 Minuten Unterbrechung. Dauerhafte Timer (Rebuild, Erinnerungen) und Oban-Jobs überstehen das.
- Ziel des Backups: Hetzner Storage Box BX11 (oder Backblaze B2), restic-Verschlüsselung mit Passwort, das nur der Nutzer und `secrets.json` kennen.
- Aufbewahrung: 7 täglich, 4 wöchentlich, 6 monatlich.
- **Wiederherstellungsprobe monatlich, automatisch:** restic-Restore in ein getrenntes Compose-Projekt mit freien Ports (`docker compose -p comma-restore-test …`), Start, Prüfung: Login möglich (Test-Admin-Pfad), Anzahl Conversations und Gedächtnis-Fakten wie im Original, dann Abbau. Ergebnis im Dashboard (`15`), Alarm bei Fehler.
- Windows-VPS: wöchentlicher Snapshot beim Anbieter. Das Chrome-Profil ist mit DPAPI an den Windows-Benutzer gebunden; ein Datei-Backup allein ließe sich auf einer anderen Maschine nicht entschlüsseln. Deshalb Snapshot der ganzen Maschine.
- Abnahmetest 5 aus `16` enthält eine Wiederherstellung auf eine leere Maschine.

---

## 6. Updates und Upstream

- Deployt wird der `main` des Forks `mika2709/comma` auf dem Linux-VPS: `git pull`, `docker compose up -d --build`. Das bestehende Vorgehen aus `DEPLOYMENT.md` "Upgrade and recovery" gilt (vorher Backup).
- Upstream AFK-surf/Comma wird regelmäßig (wöchentlich) in den Fork gemergt, als eigener PR, mit Konfliktlösung und Tests. Die eigenen Änderungen sind modular gehalten (eigene Module, Konfigurationsschalter), damit Konflikte klein bleiben.
- Hindsight-Upstream per `git subtree pull` in `services/hindsight` (`04`).
- Die Regeln in `AGENTS.md` zu Staging/Produktion von AFK-surf betreffen dieses Deployment nicht.

---

## 7. Laufende Kosten (Schätzung)

| Posten | pro Monat |
|---|---|
| Linux-VPS Hetzner CPX42 | etwa 69 € |
| GPU-Server Hetzner GEX44 (nur wenn README Frage 22a mit "GPU dazu" beantwortet) | etwa 200 € |
| Windows-VPS Contabo VPS 30 mit Lizenz | etwa 20 bis 35 € |
| Backup-Speicher | etwa 4 € |
| Apple Developer | etwa 8 € (99 $ pro Jahr) |
| Hauptmodell | nutzungsabhängig, der größte Posten |
| Crew, Erfasser, Auditor, Verifier | wenige Euro bis niedrig zweistellig |
| Entscheidungsmodelle | unter 1 € |
| Embeddings, Reranker | unter 2 € |
| GPT-Live | nutzungsabhängig |

Der Nutzer will Qualität vor Kosten (E84).

---

## 8. Abnahme

- `docker compose up -d --build` läuft auf einem frischen Ubuntu-VPS ohne Handgriffe durch.
- Selbsthost-Rauchtest aus `DEPLOYMENT.md` ("Verify an isolated installation") grün, plus: Gerät des Nutzers lässt sich anbinden, Browser über `LocalCDP` öffnet eine Seite, Gedächtnisdienst gesund, ein Entscheidungs-Call pro Anbieter klappt.
- Backup und automatische Wiederherstellungsprobe laufen.
