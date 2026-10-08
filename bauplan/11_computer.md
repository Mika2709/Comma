# 11 Computer: eigener Windows-Rechner und der Rechner des Nutzers

Ziel: Der Assistent hat einen eigenen, dauerhaft laufenden Windows-Rechner in der Cloud mit echtem Desktop und Browser, in dem Logins erhalten bleiben (E75 bis E79). Er steuert ihn mit mehreren Methoden: Koordinaten als Hauptweg, Bedienelemente-Baum zum Lesen und Prüfen, APIs wo vorhanden. Bei Bedarf kommt er an den Rechner des Nutzers. Kein Linux-Desktop.

Gewichtung: Das ist ein Puzzleteil, kein Kernthema (E8). Fertige Bausteine nutzen, wenig eigener Code.

Pfad-Abkürzungen: `sa/` = `systems/apps/`, `cn/` = `systems/connector/salix-connect/`.

---

## Ist-Zustand (Stand `ba4f55d`)

- Browser nur über Cloudflare Browser Run per CDP. Seam: `Application.get_env(:salix_agent, :browser_provider, SalixAgent.Browser.Cloudflare)` (`sa/salix_agent/lib/salix_agent/browser.ex:306-307`), Callbacks create/close/status/connection (`browser/cloudflare.ex:5-35`). Session endet nach 60 s Leerlauf. Gespeichert werden nur Cookies und First-Party-localStorage, höchstens 1 MiB (`browser/storage.ex:2,39-46`). Ein aktiver Browser pro Group (`sa/salix_store/lib/salix_store/browser_storage.ex:34-44`).
- Browser-Steuerung: Refs aus `querySelectorAll`, die ersten 100 interaktiven Elemente (`sa/salix_agent/lib/salix_agent/browser/commands.ex:345`), x,y-Klicks, JPEG-Screenshots, Übernahme durch den Nutzer, Unknown-Outcome-Sperre (`browser_bindings.ex:215`).
- Browser-Settings verlangen eine 32-hex `account_id` plus Token (`browser_settings.ex:85-89,105-107`).
- Desktop: nur macOS über `salix-connect` und einen Swift-Helfer. Beim Agenten kommt nur der Pixel-Modus an. Der Helfer kann Accessibility (`ComputerUseCLI.swift:146-190`, `CommaComputerUseDaemon/main.swift:75-82`), der Go-Connector lehnt es ab (`cn/computer_use_params.go:183-184`). darwin-Sperre (`cn/computer_use.go:21-22`, `computer_use_cli.go:31-49`), Unix-Socket-JSON-Protokoll (`computer_use.go:29-52`).
- MCP: `salix_mcp` kann `stdio` und `streamable-http` (`sa/salix_mcp/lib/salix_mcp/store.ex:593,609`, `install.ex:149`). Server-seitige MCP-URLs auf private Adressen (dazu gehört der Tailscale-Bereich 100.64.0.0/10) werden abgelehnt (`sa/salix_mcp/lib/salix_mcp/url_policy.ex:294-308`), außer exakt freigegebene `http://127.0.0.1`/`localhost`-URLs (`:60-70`) oder global `allow_private_http_targets`.
- Selbsthost: Connector-Installation scheitert ohne Release-Artefakte (`02`).

---

## 1. Architektur

```
Linux-VPS (Comma-Server, Docker)  ──Tailscale──  Windows-VPS (Rechner des Assistenten)
   SalixAgent.Browser.LocalCDP  ───────────────▶  Chrome, eigenes Profil, CDP :9222
   salix_mcp (streamable-http)  ───────────────▶  Windows-MCP :8765 (Desktop, UIA, PowerShell, Dateien)
                                 ──Tailscale──  Rechner des Nutzers (macOS: salix-connect; Windows: Windows-MCP)
                                 ──Tailscale──  iPhone, Laptop (RDP zur Übernahme)
```
(Strukturbild, kein Code.)

- Kein eigener Windows-Port des Go-Connectors. Desktop-Steuerung kommt von Windows-MCP (CursorTouch, MIT, Version 0.8.7 vom 2026-09-30), angebunden als MCP-Server über Streamable HTTP. Begründung: fertiger, gepflegter Server mit Koordinaten-Klicks, UI-Automation-Snapshot, Screenshot, Tastatur, PowerShell, Dateien und Bearer-Auth. Cuas `cua-computer-server` ist abgekündigt und hat keine Auth; Cua Driver hat noch keine Fernschnittstelle.
- Browser über CDP direkt aus Comma (neuer Provider `LocalCDP`), weil Comma dafür schon Werkzeuge, Screenshots, Übernahme und die Unknown-Outcome-Sperre hat.
- Browser und Desktop sind derselbe Rechner und dasselbe Lease (`08` Abschnitt 3).

---

## 2. Windows-VPS einrichten

Anbieter: Contabo Cloud VPS 30 (8 vCPU, 24 GB RAM) mit Windows Server 2022 Datacenter, Lizenz von Contabo. Wechselregel: Ist der Rundweg "Screenshot holen, klicken, neuer Screenshot" im Mittel über 2 Sekunden oder ruckelt der Desktop, Umzug auf AWS Lightsail Windows 16 GB (Lizenz inklusive, etwa 124 $ pro Monat, Region Frankfurt).

Schritte (als Runbook in `bauplan/` oder später in `docs/`, im Doku-Budget):
1. Windows-Updates, Zeitzone `W. Europe Standard Time`, Region de-DE, Tastatur Deutsch. Anzeigesprache Deutsch.
2. Lokaler Standardbenutzer `agent` (kein Admin). Automatische Anmeldung über Sysinternals Autologon (speichert das Passwort als LSA-Secret).
3. Energie: nie Standby, kein Bildschirmschoner, keine Sperre bei Inaktivität. Auflösung 1920×1080, Skalierung 100 Prozent (Koordinaten bleiben stabil).
4. Tailscale als Dienst, Tag `tag:agent-pc`. Windows-Firewall: RDP (3389) nur über die Tailscale-Schnittstelle, öffentlich gesperrt.
5. Chrome stable. Scheduled Task "Agent Chrome" bei Anmeldung von `agent` (interaktive Sitzung, nicht Session 0): `chrome.exe --remote-debugging-port=9222 --user-data-dir=C:\agent\chrome-profile --remote-allow-origins=<Origin des Comma-Clients>`. Seit Chrome 136 wirkt der Debug-Port nur mit eigenem Profilordner; das ist gewollt. `--remote-debugging-address` wirkt nicht mehr, Chrome bindet nur an 127.0.0.1.
6. `tailscale serve --tcp=9222 tcp://localhost:9222` (nur im Tailnet sichtbar).
7. Python 3.13, `windows-mcp==0.8.7` (gepinnt). Scheduled Task "Agent MCP" bei Anmeldung von `agent`: `windows-mcp serve --transport streamable-http --host 127.0.0.1 --port 8765 --auth-key <Schlüssel>`, Umgebung `ANONYMIZED_TELEMETRY=false`. Danach `tailscale serve --tcp=8765 tcp://localhost:8765`.
8. RDP-Trennung: Wird eine RDP-Sitzung getrennt, rendert Windows den Desktop nicht mehr (schwarze Screenshots). Scheduled Task auf Ereignis "Sitzung getrennt" (Microsoft-Windows-TerminalServices-LocalSessionManager/Operational, Ereignis 24) führt als Admin `tscon <Sitzungs-ID> /dest:console` aus. Danach Auflösung prüfen (kann auf 1024×768 fallen) und per Skript wieder auf 1920×1080 setzen.
9. Tailscale-ACL: Port 9222 und 8765 auf `tag:agent-pc` nur von `tag:comma-server`. RDP nur von Geräten des Nutzers.
10. Erster Login-Durchgang: Der Nutzer meldet sich per RDP einmal im Agent-Chrome bei seinen Diensten an (Google, Amazon, Behörden-Portale, was er nutzt), inklusive 2FA, und hinterlegt seine virtuelle Karte mit Limit (`10` Abschnitt 7). Passwörter speichert Chromes Passwortmanager im Agent-Profil, der Assistent sieht sie nie.

Abnahmeprüfung des Rechners: nach RDP-Trennung ein Screenshot über Windows-MCP ist nicht schwarz und hat 1920×1080; nach Neustart des VPS laufen Chrome und Windows-MCP ohne Eingriff.

Wenn Windows-MCP auf Windows Server nicht sauber läuft (die Doku listet offiziell Windows 7 bis 11): Ausweichweg ist Cua Driver (`cua-driver serve` als Scheduled Task in der interaktiven Sitzung) mit einer kleinen Brücke, die seine Named Pipe `\\.\pipe\cua-driver` als Streamable-HTTP-MCP-Server mit Bearer-Auth im Tailnet anbietet. Gleiche Werkzeuge (Fensterzustand mit UIA-Baum und Bild, Klick per Element-Index oder Pixel, Tippen, Tasten).

---

## 3. Comma: Browser über `LocalCDP`

- Neues Modul `SalixAgent.Browser.LocalCDP` mit den vier Callbacks des Providers (create, close, status, connection). Ziel: `http://<Tailscale-IP des Windows-VPS>:9222`. Verbindung immer über die IP, nicht über einen MagicDNS-Namen (Chrome prüft den Host-Header auf IP oder `localhost`). Die von `/json/version` gelieferte `webSocketDebuggerUrl` enthält `127.0.0.1`; der Provider ersetzt Host und Port durch die Tailscale-Adresse.
- `close` schließt nur die Tabs des Agenten, nie Chrome selbst.
- Konfiguration: `config :salix_agent, browser_provider: SalixAgent.Browser.LocalCDP` plus `local_cdp_url`. `browser_settings.ex` bekommt eine Variante ohne die Cloudflare-Pflichtfelder `account_id` und Token (`:85-89,105-107`).
- Die Cookie-/localStorage-Versiegelung (`browser/storage.ex`) ist bei `LocalCDP` aus: Das Profil auf dem Windows-Rechner ist selbst dauerhaft (IndexedDB, Service Worker, alles).
- **Lesen über den Accessibility-Baum:** Snapshots auf CDP `Accessibility.getFullAXTree` umstellen statt `querySelectorAll` mit den ersten 100 Elementen (`commands.ex:345`). Refs zeigen auf AX-Knoten mit Rolle, Name, Zustand (checked, expanded, disabled), Wert und Begrenzungsrahmen. Lange Seiten werden in Abschnitten geliefert.
- **Klicken:** Koordinaten bleiben der Hauptweg. Klick per Ref ist ein Klick auf die Mitte des Rahmens des Knotens.
- **Prüfen nach dem Klick:** neuer Screenshot oder AX-Abfrage des Zielknotens, je nach Lage.
- Aktions-Gate: Zielauflösung vor jedem Klick, jeder Enter-Taste und jeder Formular-Absendung (`10` Abschnitt 3).
- Übernahme durch den Nutzer: bestehender Mechanismus (Streaming, `browser_bindings.ex` `claim`), zusätzlich jederzeit per RDP.

---

## 4. Comma: Desktop über Windows-MCP

- Registrierung als MCP-Server der Group: Transport `streamable-http`, URL `http://<Tailscale-IP>:8765/mcp`, Bearer-Key in den MCP-Secrets (`sa/salix_mcp/lib/salix_mcp/secrets.ex`).
- `url_policy.ex`: Die Allowlist `private_http_target_allowlist` (`:60-70`) akzeptiert heute nur exakte URLs mit Host `127.0.0.1` oder `localhost`. Erweitern auf exakte URLs mit beliebigem Host im Bereich 100.64.0.0/10 (Tailscale). Kein globales `allow_private_http_targets`.
- Werkzeuge, die der Agent nutzt: `Snapshot` (UIA-Baum mit Element-IDs und Koordinaten, zum Lesen und Prüfen), `Screenshot` (schnell, für den Blick), `Click` (Koordinaten, Hauptweg), `Type`, `Shortcut`, `Scroll`, `Wait`/`WaitFor`, `App` (Programme starten), `FileSystem`, `PowerShell`, `Clipboard`.
- Effektklassen für das Aktions-Gate (`10`):
  - `Snapshot`, `Screenshot`, `Wait`: `read`.
  - `Click`, `Type`, `Shortcut`: `dynamic`. Zielauflösung über `Snapshot`: Element, dessen Rahmen den Klickpunkt enthält, plus Fenstertitel und Prozess. Zweistufiger Ablauf (beschreiben, entscheiden, ausführen) wie in `10` Abschnitt 3.
  - `PowerShell`: `dynamic`, Shell-Regeln aus `10` (PowerShell-Muster).
  - `FileSystem`: Schreiben unter `C:\agent\work\` ist `internal`, sonst `external_reversible`. Löschen außerhalb von `C:\agent\work\` ist `external_irreversible`.
  - `Registry`, `Process` (beenden): `external_reversible`.
- Werkzeuge, die Windows-MCP sonst anbietet und die der Agent nicht braucht, werden in der MCP-Konfiguration ausgeblendet.

### Prompt-Regeln zur Methodenwahl (Skill `computer-use-windows`)
- Gibt es eine API oder einen Connector (Gmail, Kalender, Drive), nimm die API. Kein Klicken.
- Im Browser: Browser-Werkzeuge von Comma (AX-Baum plus Koordinaten). Auf dem Desktop: Windows-MCP.
- Klicken per Koordinaten aus dem aktuellen Screenshot.
- Text lesen über den Baum (AX oder UIA), nicht vom Bild abschreiben.
- Zustände (Häkchen, ausgewählt, ausgegraut) über den Baum prüfen.
- Vor jedem Klick auf eine Seite, die sich bewegt haben könnte, einen frischen Screenshot.
- Was außerhalb des sichtbaren Bereichs liegt: erst scrollen, dann klicken.
- Tippen über `Type`, nie Zeichen für Zeichen klicken.
- Nach jeder Aktion mit Wirkung prüfen, ob sie gewirkt hat.

---

## 5. Rechner des Nutzers

Die Antwort auf Startfrage 15 (`README.md`) entscheidet:

**macOS**
- Bestehender Weg über `salix-connect` (selbst gebaut, Start mit `--connector-token`, siehe `02` Selbsthost-Fix 2) und den Swift-Helfer.
- Accessibility-Modus freischalten: `cn/computer_use_params.go:183-184` lässt die AX-Aktionen des Helfers durch (`ComputerUseCLI.swift:146-190` bzw. `CommaComputerUseDaemon/main.swift:75-82`). Damit bekommt der Agent auch auf dem Mac Koordinaten plus Baum.

**Windows**
- Windows-MCP wie auf dem VPS, als zweiter MCP-Server "Rechner des Nutzers", im Tailnet, mit eigenem Key. Läuft als Scheduled Task in der Sitzung des Nutzers.

**Regeln für beide**
- Der Nutzer arbeitet selbst an diesem Rechner. Der Assistent nutzt ihn nur, wenn der Nutzer es verlangt oder eine Aufgabe nur dort geht (lokale Dateien, lokal installierte Programme).
- Sitzungsfreigabe: Der erste Zugriff pro 30 Minuten fragt über das Aktions-Gate ("Darf ich kurz an deinen Rechner?"). Danach gilt das Gate für unumkehrbare Aktionen wie überall. Grundlage: `computer_use_start` (`sa/salix_agent/lib/salix_agent/tools/async_ops.ex:168-190`), heute freiwillig, wird für diesen Rechner Pflicht (`env_dispatch.ex:212-217` prüft es).
- Native Coding-Worker laufen auf diesem Rechner nicht (`10` Abschnitt 3).

---

## 6. Abnahme

- Tests: `LocalCDP`-Provider gegen einen lokalen Chromium im Test (Testcontainer oder lokal gestartet), Host-Umschreibung der WebSocket-URL, AX-Snapshot mit mehr als 100 Elementen, `close` lässt Chrome laufen, URL-Policy erlaubt nur die exakte Tailnet-URL, Effektklassen der Windows-MCP-Werkzeuge.
- Manuell: Abnahmetest 11 aus `16` (Login bleibt eine Woche erhalten). Ein Kauf-Button auf einer echten Shop-Seite löst die Freigabe aus (Abnahmetest 10).
