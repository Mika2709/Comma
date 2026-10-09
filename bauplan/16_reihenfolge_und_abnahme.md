# 16 Reihenfolge, Parallelisierung und Abnahme

Der Code für fast jeden Punkt ist mit AI eine Sache von Minuten bis Stunden. Zeit kostet das Prüfen in der echten Welt: Apple-Setup, Logins im dauerhaften Browser, echte Freigaben, Gedächtnisqualität über mehrere Tage, Lean neu bauen. Plane danach.

`AGENTS.md`: höchstens ein zusätzlicher Worktree pro parallel laufendem PR. Ein PR pro Arbeitspaket unten. Titel nach `feat:`/`fix:`/`refactor:`.

---

## Phase 0: Lauffähig (seriell, zuerst)

1. **Zuerst fragen** (`README.md`). Alle Zugänge einsammeln.
2. **Selbsthost reparieren und auf dem Linux-VPS starten** (`02`): Build-Fixes, Connector-Artefakte, Browser-Secret, `decide`- und `apns`-Keys in `configure.py`, Tailscale, Domain, HTTPS.
3. **Windows-VPS einrichten** (`11` Abschnitt 2): Tailscale, Autologon, Chrome mit eigenem Profil und Debug-Port, Windows-MCP, RDP-Trennung. Der Nutzer meldet sich einmal per RDP in seinen Diensten an.
4. **Backup einrichten** (`02` Abschnitt 5): Storage Box, restic, Nachtjob 04:30, Wartelogik auf Oban-Jobs. Einmal sofort sichern und die Wiederherstellungsprobe einmal von Hand auslösen.
5. **Hauptmodell einrichten**, Kontextfenster und Cache-TTL im Modell-Template setzen (`03` Abschnitt 5). Bei einem Anthropic-Modell sofort Header und `drop_block` setzen (`03` Abschnitt 7, Schritt 1), damit der Ist-Zustand nicht mit Fehler 400 scheitert.
6. Jede Modell-ID aus `02` Abschnitt 3 und `14` Abschnitt 5 mit einem echten Call prüfen und pinnen, Ausweichregel bei Fehlschlag (`14` Abschnitt 5).
7. Rauchtest: Nachricht in Home, Antwort kommt; ein Helfer läuft; Gmail-Trigger kommt an.

Ergebnis: Comma läuft im Ist-Zustand stabil beim Nutzer.

---

## Phase 1: Fundament (parallel)

Diese Pakete laufen parallel in getrennten Worktrees. Ausnahmen: P10 wartet auf P7, P9 auf P8, der Modellteil von P5 auf P1.

| Paket | Datei | Inhalt | Hängt ab von |
|---|---|---|---|
| P1 Entscheidungs-Router | `14` | Adapter, Konfiguration pro Einsatz, `decision_log`, Schatten, Kalibrierungs-Jobs | Phase 0 |
| P2 Gedächtnisdienst | `04` | Hindsight-Fork, Schema, Herkunft, Gültigkeit, Zeit, Modelle, Comma-Client, Migration der `/memory`-Dateien | Phase 0 |
| P3 Proaktivitäts-Fixes | `07` | V39, V50, Budgets raus, Filter, Monitoring immer an | Phase 0 |
| P4 Sprache und Zeit | `12` | `de`-Locale überall, Zeit pro Turn, Zeitzone | Phase 0 |
| P5 Aktions-Gate | `10` | Gate an zentraler Stelle, Regeln, Belege, Klickprüfung Browser | Phase 0 (Modellteil braucht P1, Regeln nicht) |
| P6 Computer | `11` | `LocalCDP`-Provider mit AX-Baum, Windows-MCP als MCP-Server, URL-Policy, Effektklassen, Leases | Phase 0 |
| P7 Unsichtbare Helfer | `08` | Task-Karten weg, Abnahme aus, Stillstandserkennung (V25), Abbrechen | Phase 0 |
| P8 Eval-Rahmen und Dashboard | `15` | `mix assistant.eval` mit Szenario-Format und Richter, alle Suiten als Gerüst mit ersten Fällen, Dashboard-Seite "Assistent", Debug-Ansicht Gedächtnis, Alarme | Phase 0 |
| P9 Stabiler Präfix und Thinking-Bindung | `03` Abschnitt 7 (Schritte 1, 2, 4, 5) | Messung, Plattform-Hinweis ins Delta, `turn_scoped`-Reminder und Miniskills speichern, Anhänge stabil, Snapshot nur beim Rebuild, Feld `thinking_cutoff_id` in jedem Compaction-Commit (auch der heutigen Compaction), Dashboard-Zähler. Kernel-PR mit Beweisen. | P8 (Evals, Dashboard) |
| P10 Todo-Board und UI | `09` | Board-Projektion, CalendarItem `Task` erweitern, Board-Tools, rechte Leiste, Vorschläge integriert, Picker, Vorlagen, Server-Zustand, Web-Widgets mit Checkliste | P7 |

P10 steht in Phase 1, weil Rebuild (R8, Prüfliste), Hidden Helper (Zusagen als Board-Items) und Weck-Gate (Board-Todo, Briefing) es brauchen.

---

## Phase 2: Verhalten (parallel, nach Phase 1)

| Paket | Datei | Inhalt | Eval-Suite (`15`) | Hängt ab von |
|---|---|---|---|---|
| P11 Rebuild | `03` | Lean-Auslöser, Crew, Verifier, Kernblock (`03` Abschnitt 2), Keep-alive, Thinking im behaltenen Ende (`03` Abschnitt 7, Schritt 3) | "Rebuild", "Gedächtnis über Zeit" | P2 (Gedächtnis-Schreiber), P8, P9, P10 (R8, Prüfliste), P1 (nur für Labels) |
| P12 Hidden Helper | `05` | Vor-Turn-Abruf, Einspielen, Auditor, Gedächtnis für Worker | "Gedächtnis über Zeit", Fall 13 aus "Antwortverhalten" | P1, P2, P8, P10 |
| P13 Antwortform | `06` | `TurnFormSelector`, Sende-Tool, Reaktionen, `opening` weg | "Antwortverhalten" | P1, P8, P9 (Katalog über Migrationshinweis) |
| P14 Weiche Delegation | `08` | Wert `delegation` in der Antwort-Richtung, Regeln im Prompt | "Delegation" | P1, P8, P13 (gleicher Vor-Turn-Call) |
| P15 Weck-Gate | `07` | Gate vor der Aufmerksamkeitsprüfung, harte Regeln, Stufen aus `14` | "Proaktivität" | P1, P2 (Personen aus dem Gedächtnis), P3, P8, P10 |

Ein Paket ist erst fertig, wenn seine Eval-Suite grün ist.

P11 ändert den Lean-Kern. Plane dafür einen eigenen PR, der nur Kernel-Änderungen plus Beweise enthält, TLA unverändert mit Begründung im PR (`03` Abschnitt 1, Punkt 11), und einen zweiten PR für die Elixir-Seite (Crew, Verifier). Für P9 gilt dasselbe.

---

## Phase 3: Geräte und Panels (parallel, Abhängigkeiten in der Tabelle)

| Paket | Datei | Inhalt | Hängt ab von |
|---|---|---|---|
| P16 iPhone | `13` | Push für Chat und Meldungen, Anwesenheit für `ios` und `web`, Board, Reaktionen, Picker nativ, Widgets im `WKWebView` | P10, P13 (Reaktionen) |
| P17 Voice | `13` | Voice-to-Voice in Electron und iPhone über die vorhandene Sprach-Schnittstelle | P16 für den iPhone-Teil |
| P18 Eigener Rechner | `11` | Lokale Steuerung des Nutzerrechners (macOS: `salix-connect` plus Accessibility-Modus; Windows: Windows-MCP wie auf dem VPS) | P6 |
| P19 Anheftbare Panels | `09` Abschnitt 3, "Weitere Panels" | Widget mit Zustand an die rechte Leiste anheften, `panel_id` im `ConversationGroupActor`, `ui.create` mit `panel_id` | P10, P16 |

---

## Phase 4: Einschwingen (eine bis zwei Wochen echte Nutzung)

- Alle Entscheidungsmodelle laufen im Schatten, Labels sammeln sich.
- Evals nächtlich (`15`). Nach 7 Tagen: weiche Prüfgrenze (`06`), Reranker-Schwellen (`05`), Stufen pro Einsatz (`14` Abschnitt 2), Rebuild-Schwelle (`03`) anhand der Zahlen justieren.
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
14. **Panels:** Ein Widget wird angeheftet. Sein Zustand ist auf Desktop und iPhone gleich, und der Router liest ihn im nächsten Turn.
15. **Thinking-Bindung** (nur bei Anthropic-Hauptmodell): Eine Woche Alltag ohne Fehler 400 und ohne `prefix_binding_mismatch` im Dashboard, außer direkt nach einer Tool-Änderung.
