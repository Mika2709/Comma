# Bauplan: persönlicher Assistent auf Comma-Basis

Stand: 2026-10-09. Basis-Commit: `ba4f55d` (AFK-surf/Comma main, identisch mit dem Fork `mika2709/comma`).

Dieser Ordner ist die vollständige Bauanleitung. Eine AI, die Comma kennt, baut damit alles fertig. Alle Produktentscheidungen sind getroffen. Offene Punkte gibt es nur dort, wo der Nutzer Zugangsdaten oder Konten liefern muss (Abschnitt "Zuerst fragen").

Der Plan enthält keinen Code. Er nennt Ziel, Verhalten, Datenmodell, Andockstellen im Code (Datei:Zeile beim Stand `ba4f55d`), Reihenfolge und Abnahme. Zeilennummern können driften. Prüfe jede Stelle vor der Änderung im echten Code.

---

## Zuerst fragen (bevor du Code schreibst)

Stell dem Nutzer diese Fragen in einer einzigen Nachricht. Kurz, ohne Erklärungen, die er nicht braucht. Wenn er etwas noch nicht hat, nenn ihm den einen Link oder Schritt, um es zu bekommen.

**Konten und Schlüssel**
1. Hauptmodell: Welcher Anbieter und welches Modell für den Hauptchat (Router) und die Helfer (Worker)? Empfehlung: Claude Opus 5.5 (`claude-opus-5-5`) über die Anthropic-API, oder ein vorhandenes Claude- oder Codex-Abo (Comma unterstützt Abo-Konten). API-Key oder Abo-Login.
2. Together-AI-API-Key (DeepSeek V4.1 Flash für Cleanup-Crew, Auditor, ersten Verifier und als Entscheidungsmodell-Kandidat).
3. OpenRouter-API-Key (Entscheidungsmodelle Jev, Luna Decisions, Clef-flash, Liquid d1 und das Modell des zweiten Verifiers).
4. Cloudflare-Konto mit API-Token für Workers AI (Clef 27B für das Aktions-Gate). Siehe `02_infrastruktur_und_selfhost.md` für weitere Dienste über diesen Token.
5. Anbieter-Key für Embeddings und Reranker (siehe `04_gedaechtnis.md`, Abschnitt Modelle).
6. Composio-API-Key und verbundenes Gmail-Konto (E-Mail-Weckung). Kalender, Drive, Slack nur, wenn er sie nutzt.
7. Exa-API-Key (Websuche der Agenten), falls er Websuche will. Empfehlung: ja.
8. OpenAI-API-Key für Voice-to-Voice mit GPT-Live.
9. Telegram-Bot-Token (Ausweichkanal für Meldungen, wenn die App zu ist). Optional, empfohlen.
10. Apple-Developer-Konto (Team-ID, APNs-Key `.p8`, Key-ID) für die iPhone-App mit Push. Ohne Konto läuft das Handy nur über Telegram.

**Maschinen**
11. Linux-VPS für den Comma-Server: Anbieter, IP, SSH-Zugang. Mindestens 8 vCPU, 16 GB RAM, 200 GB SSD.
12. Windows-VPS mit echtem Desktop (RDP) für die Computersteuerung des Assistenten: IP, RDP-Zugang. Mindestens 4 vCPU, 16 GB RAM.
13. Domain für HTTPS (Comma braucht vier Origins unter einer Domain, siehe `02`). DNS-Zugang.
14. Tailscale-Konto (privates Netz zwischen beiden VPS, seinem Rechner und dem Handy).

**Persönliche Fakten**
15. Betriebssystem seines eigenen Rechners: macOS oder Windows. Davon hängt die lokale Steuerung ab (`11_computer.md`).
16. Zeitzone. Annahme: `Europe/Berlin`.

**Bestätigen (von Claude für ihn entschieden, weil er keine offenen Fragen wollte)**
17. Menschliche Abnahme von Helfer-Ergebnissen ist standardmäßig aus. Schutz kommt nur über das Aktions-Gate für unumkehrbare Aktionen.
18. Beide proaktiven Budgets fallen ersatzlos weg. Es bleibt nur ein Schleifenschutz, der identische Wiederholungen desselben Ereignisses blockiert, nie unterschiedliche Ereignisse.
19. Hindsight läuft als eigener, geforkter Dienst neben Comma (nicht nach Elixir portiert).

Wenn er bei 17 bis 19 widerspricht, gilt seine Antwort. Trag sie in `01_zielbild_und_entscheidungen.md` ein.

---

## Wie du mit dem Nutzer redest

Das ist Teil des Produkts und gilt auch für dich während des Baus.
- Deutsch. Englische Fachwörter dürfen bleiben.
- Kurz, ohne Filler, aber vollständig. Viele Bullet Points sind nicht kurz.
- Eine Antwort pro Runde, keine Aufteilung in mehrere Nachrichten.
- Lösung nennen statt Überlegungen hinwerfen. Wenn du die Antwort weißt, sag sie.
- Zustimmung mit Einschränkung in einem Satz: "Ja, aber X, weil Y."
- Kein Ja-Sager, kein Nein-Sager. Echte Einwände nennen, sonst umsetzen.
- Fachbegriffe beim ersten Gebrauch in einem Halbsatz erklären.
- Kleinkram (Build-Fixes, Konfiguration) nicht erklären, einfach erledigen und im PR festhalten.
- Keine voreiligen Annahmen über Anbieter. Nichts hart an Anthropic koppeln.
- Aufwand mit AI-Coding und parallelen Worktrees schätzen (Minuten bis Stunden), nicht klassisch.

---

## Lesereihenfolge

| Datei | Inhalt |
|---|---|
| `01_zielbild_und_entscheidungen.md` | Was der Assistent am Ende tut, alle verbindlichen Entscheidungen, was bewusst nicht gebaut wird |
| `02_infrastruktur_und_selfhost.md` | VPS-Aufbau, Netz, Dienste, Modelle und Anbieter, Selbsthost-Reparaturen, Backups |
| `03_kontext_und_rebuild.md` | Endlos-Chat: Rebuild bei Cache-Ablauf, 100k-Schwelle, Notbremse, Cleanup-Crew, Verifier, Lean-Änderungen |
| `04_gedaechtnis.md` | Gedächtnisdienst (Hindsight-Fork), Datenmodell, Gültigkeit, Herkunft, Zeit, Suche, Erfassung |
| `05_hidden_helper.md` | Abruf vor jedem Turn, Einspielen, Auditor nach dem Turn, Gedächtnis für Helfer |
| `06_antwortform_und_reaktionen.md` | Antwortform-Entscheidung, Sende-Tool, mehrere Nachrichten, Reaktionen |
| `07_proaktivitaet.md` | Ereignisquellen, Weck-Gate, Budgets raus, Bugfixes V39/V50, Zustellung |
| `08_delegation_und_worker.md` | Unsichtbare Helfer, weiche Delegation, Abnahme aus, Stillstandserkennung, Abbrechen |
| `09_todo_board_und_ui.md` | Todo-Board rechts, Vorschläge integriert, Datum-/Uhrzeit-Picker, wiederverwendbare UI, Zustand auf dem Server |
| `10_freigaben.md` | Hartes Aktions-Gate über alle Pfade, Regeln plus Modell, Belege, Klickprüfung |
| `11_computer.md` | Windows-VPS-Desktop, Browser mit dauerhaftem Profil, eigener Rechner, Koordinaten plus UI-Baum |
| `12_sprache_und_zeit.md` | Deutsch überall, Zeit in jedem Turn, Zeitzone |
| `13_handy_und_voice.md` | iPhone-App mit Push, Telegram als Ausweichkanal, Voice-to-Voice |
| `14_entscheidungsmodelle.md` | Entscheidungs-Router, Anbieter, Einsatzzwecke, Schattenbetrieb, Kalibrierung |
| `15_tests_evals_betrieb.md` | Tests nach AGENTS.md, Evals für Verhalten, Beobachtbarkeit |
| `16_reihenfolge_und_abnahme.md` | Phasen, parallele Worktrees, Abhängigkeiten, Abnahmetests |
| `17_report_abgleich.md` | Abgleich mit dem konsolidierten Report (F01-F29, V01-V76): erledigt, eingeplant, bewusst später |

---

## Regeln für die umsetzende AI

- Halte dich an `AGENTS.md` im Repo-Root. Die wichtigsten Punkte für diesen Plan:
  - Neue Domänen-Entitäten nur, wenn ein bestehendes Konzept nicht reicht. Dann `docs/architecture/DOMAIN_CONCEPTS.md` im selben PR ergänzen (Identität, Owner, Lebenszyklus, Beziehungen, Code-Links).
  - Salix-Nachrichten und Teilnehmer nur über `ConversationServer -> ConversationActor` ändern.
  - `docs/` außer `docs/architecture/` und `docs/user_manual/` hat ein Budget: höchstens 20 Dateien, je höchstens 20.000 Bytes. Nach Doku-Änderungen `pnpm docs:check`. Dieser `bauplan/`-Ordner liegt bewusst außerhalb von `docs/`.
  - Native Fähigkeiten zwischen Electron-Main und Renderer nur über generierte Leaves in `clients/packages/native-bridge/src/capability-leaves.ts`.
  - TLA+-Modelle unter `tla/` mitziehen, wenn sich modellierte Übergänge ändern. `make tla` ausführen.
  - Tests nach Verhalten, nicht nach Implementierungstext. Fixes brauchen Regressionstests.
  - PR-Titel: `feat: ...`, `fix: ...`, `refactor: ...`.
- Die Skill `ste-writing`, auf die `AGENTS.md` verweist, liegt nicht in diesem Checkout. Schreib Doku in kurzen, aktiven Sätzen.
- Der Nutzer betreibt sein eigenes Deployment. Die Regeln in `AGENTS.md` zu Staging und Produktion von AFK-surf betreffen ihn nicht. Er deployt seinen Fork-Main auf seinen VPS (`02`).
- Halte Änderungen modular (eigene Module, klare Schalter), damit Upstream-Updates von AFK-surf/Comma weiter einfließen können.
- Arbeite parallel in Worktrees, wo `16_reihenfolge_und_abnahme.md` es erlaubt.
- Kosten sind zweitrangig. Der Nutzer zahlt lieber 30 bis 50 Prozent mehr für einen spürbar besseren Assistenten. Spare nicht an Modell-Calls, Verifiern oder Kontext, wenn es die Qualität hebt.

---

## Begriffe

| Begriff | Bedeutung |
|---|---|
| Router | Der Hauptagent im einzigen Chat (Home). Einziger Gesprächspartner des Nutzers. |
| Worker / Helfer | Hintergrund-Agent für eine Task. Für den Nutzer unsichtbar. |
| Task | Ein Arbeitsauftrag mit eigenem Worker, Verlauf und Status (bestehende Comma-Entität). |
| Group | Der Arbeitsbereich, zu dem Router, Tasks, Geräte und Pins gehören. |
| Rebuild | Neuaufbau des Router-Kontexts (ersetzt die heutige Compaction). Erledigtes wird verdichtet, Offenes bleibt wörtlich. |
| Watermark | Grenze im Verlauf: davor steht die Summary, danach die Live-Nachrichten. |
| Cleanup-Crew | Parallele, günstige Modell-Calls mit je einer festen Aufgabe beim Rebuild. Keine sichtbaren Agenten. |
| Verifier | Prüft das Gesamtergebnis der Crew und lässt Fehler automatisch korrigieren. |
| Hidden Helper | Abruf vor jedem Turn plus Auditor nach jedem Turn. Schreibt dem Router nie direkt. |
| Entscheidungsmodell | Kleines, schnelles Modell, das zwischen festen Optionen wählt und Wahrscheinlichkeiten liefert (z.B. Jev). |
| Weck-Gate | Entscheidung, ob ein Ereignis den Router weckt. |
| Aktions-Gate | Harte Freigabe-Schranke vor unumkehrbaren Aktionen. |
| Gedächtnisdienst | Der Hindsight-Fork, der Fakten speichert, verknüpft und findet. |
