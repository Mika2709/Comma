# 16 Reihenfolge, Parallelisierung und Abnahme

Der Code für fast jeden Punkt ist mit AI eine Sache von Minuten bis Stunden. Zeit kostet das Prüfen in der echten Welt: Apple-Setup, Logins im dauerhaften Browser, echte Freigaben, Gedächtnisqualität über mehrere Tage, Lean neu bauen. Plane danach.

`AGENTS.md`: höchstens ein zusätzlicher Worktree pro parallel laufendem PR. Ein PR pro Arbeitspaket unten. Titel nach `feat:`/`fix:`/`refactor:`.

---

## Phase 0: Lauffähig (seriell, zuerst)

1. **Zuerst fragen** (`README.md`). Alle Zugänge einsammeln.
2. **Selbsthost reparieren und auf dem Linux-VPS starten** (`02`): Build-Fixes, Connector-Artefakte, Browser-Secret, `decide`- und `apns`-Keys in `configure.py`, Tailscale, Domain, HTTPS.
3. **Windows-VPS anbinden** (`11`, nur der Connector-Teil): `salix-connect` für Windows bauen, als Dienst starten, Gerät in der Group freigeben. Desktop-Steuerung kommt in Phase 1.
4. **Hauptmodell einrichten**, Kontextfenster und Cache-TTL im Modell-Template setzen (`03`).
5. Rauchtest: Nachricht in Home, Antwort kommt; ein Helfer läuft; Gmail-Trigger kommt an.

Ergebnis: Comma läuft im Ist-Zustand stabil beim Nutzer.

---

## Phase 1: Fundament (parallel)

Diese Pakete sind unabhängig und laufen parallel in getrennten Worktrees.

| Paket | Datei | Inhalt | Hängt ab von |
|---|---|---|---|
| P1 Entscheidungs-Router | `14` | Adapter, Konfiguration pro Einsatz, `decision_log`, Schatten, Kalibrierungs-Jobs | Phase 0 |
| P2 Gedächtnisdienst | `04` | Hindsight-Fork, Schema, Herkunft, Gültigkeit, Zeit, Modelle, Comma-Client, Migration der `/memory`-Dateien | Phase 0 |
| P3 Proaktivitäts-Fixes | `07` | V39, V50, Budgets raus, Filter, Monitoring immer an | Phase 0 |
| P4 Sprache und Zeit | `12` | `de`-Locale überall, Zeit pro Turn, Zeitzone | Phase 0 |
| P5 Aktions-Gate | `10` | Gate an zentraler Stelle, Regeln, Belege, Klickprüfung Browser | Phase 0 (Modellteil braucht P1, Regeln nicht) |
| P6 Windows-Desktop | `11` | Desktop-Steuerung Windows, Browser per CDP auf dem Windows-VPS, Leases | Phase 0 |
| P7 Unsichtbare Helfer | `08` | Task-Karten weg, Abnahme aus, Stillstandserkennung (V25), Abbrechen | Phase 0 |

---

## Phase 2: Verhalten (parallel, nach Phase 1)

| Paket | Datei | Inhalt | Hängt ab von |
|---|---|---|---|
| P8 Rebuild | `03` | Lean-Auslöser, Crew, Verifier, Kernfakten-Block, Keep-alive | P2 (Gedächtnis-Schreiber), P1 (nur für Labels) |
| P9 Hidden Helper | `05` | Vor-Turn-Abruf, Einspielen, Auditor, Gedächtnis für Worker | P1, P2 |
| P10 Antwortform | `06` | `TurnFormSelector`, Sende-Tool, Reaktionen, `opening` weg | P1 |
| P11 Weiche Delegation | `08` | Delegations-Flag, Regeln im Prompt | P1, P10 (gleicher Vor-Turn-Call) |
| P12 Weck-Gate | `07` | Gate vor der Aufmerksamkeitsprüfung, harte Regeln | P1, P2 (Personen aus dem Gedächtnis), P3 |
| P13 Todo-Board und UI | `09` | Board-Entität, Board-Tool, rechte Leiste, Vorschläge integriert, Picker, Vorlagen, Server-Zustand | P7 |

P8 ändert den Lean-Kern. Plane dafür einen eigenen PR, der nur Kernel-Änderungen plus Beweise und TLA-Anpassungen enthält, und einen zweiten PR für die Elixir-Seite (Crew, Verifier).

---

## Phase 3: Geräte (parallel, nach P13)

| Paket | Datei | Inhalt |
|---|---|---|
| P14 iPhone | `13` | Push für Chat und Meldungen, Board, Reaktionen, Picker nativ, `app_active` |
| P15 Voice | `13` | Voice-to-Voice in Electron und iPhone über die vorhandene Sprach-Schnittstelle |
| P16 Eigener Rechner | `11` | Lokale Steuerung des Nutzerrechners (macOS: Accessibility-Modus freischalten; Windows: gleicher Dienst wie VPS) |

---

## Phase 4: Einschwingen (eine bis zwei Wochen echte Nutzung)

- Alle Entscheidungsmodelle laufen im Schatten, Labels sammeln sich.
- Evals nächtlich (`15`). Nach 7 Tagen: weiche Prüfgrenze (`06`), Reranker-Schwellen (`05`), Rebuild-Schwelle (`03`) anhand der Zahlen justieren.
- Nach 14 Tagen: erster Kalibrierungs- und Wechsel-Lauf (`14`).
- Fehler aus dem Alltag sofort fixen, mit Regressionstest.

---

## Abnahme (fertig, wenn alles besteht)

### Fünf Alltagstests
1. **Echter Mail- oder Terminauftrag:** Mail finden, passende Terminoptionen vorbereiten, mit Picker knapp nachfragen, exakt freigeben lassen, danach genau einmal senden. Antwort im Chat nur ✉️ oder ein Satz.
2. **Korrigierte Vorliebe:** "Früher X, ab jetzt Y" gilt am nächsten Tag im Hauptchat, im Briefing und bei Helfern. Die frühere Aussage ist als ersetzt auffindbar, mit Sprung zur Originalnachricht.
3. **Handy gesperrt:** Eine wichtige neue Mail erreicht das gesperrte iPhone als Push mit dem Text des Assistenten. Unwichtiges bleibt still. Keine doppelten Meldungen.
4. **Todo auf zwei Geräten:** Ein Todo, das der Assistent anlegt, ist auf Desktop und iPhone sichtbar. Hakt der Nutzer es auf dem iPhone ab, weiß der Assistent es im nächsten Turn.
5. **Unterbrechung oder Update:** Server-Neustart mitten in einem Helfer-Job und mitten in einem Rebuild. Danach: Task- und Freigabestatus verständlich, keine doppelte Mail, Proaktivität läuft weiter ohne Home-Besuch, Gedächtnis und Browser-Logins sind noch da. Wiederherstellung aus dem Backup auf eine leere Maschine funktioniert.

### Zusätzliche Kerntests
6. **Endlos-Chat:** 30 Tage Nutzung ohne neuen Chat. Die Summary ist nicht gewachsen (Budget aus `03`). Fragen zu Dingen von Tag 1 werden richtig beantwortet.
7. **Antwortform:** Eval-Suite "Antwortverhalten" grün. Der Nutzer bestätigt nach einer Woche: keine Romane, keine fehlende Kerninfo.
8. **Unsichtbare Helfer:** Eine Rechercheaufgabe läuft im Hintergrund. Im Chat erscheint nur 🔍, dann ⏳, dann das Ergebnis und ✅. Keine Task-Karte, keine Abnahme.
9. **Hängender Helfer:** Ein absichtlich hängender Helfer wird innerhalb von 15 Minuten erkannt und vom Router behandelt.
10. **Aktions-Gate:** Ein Kauf-Button auf einer echten Shop-Seite, eine Mail mit Prompt-Injection und ein `env.exec`-Befehl mit `curl -X POST` lösen jeweils eine Freigabe aus. Ohne Freigabe passiert nichts.
11. **Windows-Rechner:** Der Assistent loggt sich auf einer Seite ein (Nutzer hilft einmal per RDP bei 2FA), eine Woche später ist er noch eingeloggt.
12. **Deutsch:** Jede Oberfläche, jedes Briefing, jede Meldung ist Deutsch.
13. **Zeit:** "Erinner mich übermorgen um 9" landet richtig, auch nach Rebuild. Der Router weiß ohne Nachfrage, wie lange die letzte Nachricht her ist.
