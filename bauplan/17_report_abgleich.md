# 17 Abgleich mit dem konsolidierten Report

Der Nutzer hat einen Report aus vier AI-Analysen geschickt (F01 bis F29 fehlende Funktionen, V01 bis V76 Verbesserungen). Viele Punkte sind Breite für Firmenkunden, nicht sein Kern. Diese Tabelle sagt für jeden Punkt: wo er im Plan steht, oder warum er bewusst später kommt.

Legende: **Plan** = wird gebaut (Datei), **teilweise** = Kern wird gebaut, Rest später, **später** = bewusst nicht in diesem Plan, **entfällt** = durch eine Entscheidung überflüssig.

## A. Fehlende Funktionen

| ID | Thema | Status |
|---|---|---|
| F01 | Automatische Memory-Extraktion | Plan `04` (Erfassung nach jedem Turn und beim Rebuild) |
| F02 | Automatischer Abruf vor Antworten | Plan `05` |
| F03 | Semantische/hybride Memory-Suche | Plan `04` (Hindsight: semantisch, BM25, Graph, Zeit, Reranker) |
| F04 | Strukturiertes Wissensmodell (Personen, Firmen, Projekte, Beziehungen) | Plan `04` (Entitäten, Links, Themenseiten) |
| F05 | Automatische Konsolidierung | Plan `03` und `04` (Crew beim Rebuild, nächtliche Konsolidierung) |
| F06 | Versionierte, zeitlich gültige Erinnerungen | Plan `04` |
| F07 | Herkunftsnachweise | Plan `04` (Originalnachricht-ID je Fakt) |
| F08 | Memory-Verwaltung für den Nutzer | später; nur Debug-Ansicht `15` (Nutzer will nichts pflegen) |
| F09 | Lernendes Kommunikationsprofil | Plan `06` Abschnitt 4 |
| F10 | Lernen aus proaktivem Feedback | Plan `07` und `14` (Labels, Kalibrierung, Vorlieben als Fakten) |
| F11 | Situatives Schweigen und Antwortumfang | Plan `06` |
| F12 | Ziel-, Projekt- und Prioritätengedächtnis | teilweise `04` (Kategorien `ziel`, `projekt`, Themenseiten) |
| F13 | Verpflichtungen erkennen und verfolgen | Plan `04` (Kategorie `zusage`) und `09` (werden Todos mit Fälligkeit) |
| F14 | Einheitliche Vorgangsidentität über Apps | später |
| F15 | App-übergreifendes Follow-up | teilweise `09` (Todos mit Wiedervorlage), Rest später |
| F16 | Autonomes E-Mail-Management | teilweise `07` (Weck-Gate, Router bearbeitet) und `10` (Senden nur mit Freigabe) |
| F17 | Konfigurierbare Autonomie- und Freigaberegeln | Plan `10` (Freigabe "einmal" oder "immer für dieses Ziel") |
| F18 | Proaktive Aufgaben direkt starten | Plan `07` (Router handelt auf versteckte Evidenz, kein Entwurf im Eingabefeld) |
| F19 | Task-Abhängigkeiten | später |
| F20 | Ruhezeiten und Unterbrechungsregeln | später als Regelwerk; Router lernt Vorlieben als Fakten (`04`), iOS-Fokus gilt automatisch |
| F21 | Stiller Modus getrennt von Monitoring-Aus | Plan `07` (Monitoring läuft immer, nur Zustellung schaltbar) |
| F22 | Geräteübergreifende Anwesenheit | Plan `13` (iPhone zählt für `app_active`) |
| F23 | Mehrere Konten derselben App | später |
| F24 | Mehrere Kalender | später |
| F25 | Persönliche Task-Manager als Quellen | später |
| F26 | WhatsApp | später |
| F27 | iPhone-Push für proaktive Meldungen | Plan `13` |
| F28 | Android | später |
| F29 | Echtes Vergessen | teilweise `04` ("vergiss das" löscht Fakt und Einspiel-Spuren; Archiv-Redaktion bleibt wie heute) |

## B. Verbesserungen

| ID | Thema | Status |
|---|---|---|
| V01 | Parallele Memory-Schreibvorgänge | entfällt weitgehend: Fakten liegen im Gedächtnisdienst, Schreiben ist serialisiert (`04`). Das Tagebuch-Append wird trotzdem atomar (`04`). |
| V02 | Memory-Suche indexieren und ranken | Plan `04` |
| V03 | Fehlgeschlagene Suchen sichtbar | Plan `04` (Ergebnis unterscheidet leer und fehlerhaft) |
| V04 | Tagebuch nach Nutzerzeitzone | Plan `12` |
| V05 | Worker-Erkenntnisse aus dem Archiv | Plan `04` (einheitliche Verlaufssuche über Router und Tasks) |
| V06 | Lange Tasks vollständig durchsuchbar | Plan `04` |
| V07 | Suche über Sitzungsgrenzen | Plan `04` |
| V08 | Semantische Suche über alte Chats | Plan `04` |
| V09 | Compaction dynamischer | Plan `03` (Rebuild bei Cache-Ablauf) |
| V10 | Token-Grenze ohne Provider-Usage | Plan `03` (lokale Schätzung für die Notbremse) |
| V11 | Microcompaction von Tool-Ausgaben | Plan `03` (Crew-Aufgabe) |
| V12 | Inhaltliche Compaction-Verifikation | Plan `03` (zwei Verifier) |
| V13 | Arbeitskontext nach Compaction-Fehler | Plan `03` (Wiederanlauf, alte Summary bleibt gültig) |
| V14 | Archivierung erneut versuchen | Plan `03` (eigener Retry-Job) |
| V15 | Metadaten jahrelanger Chats begrenzen | später |
| V16 | Antwortlänge durchsetzen | Plan `06` |
| V17 | Schnelle Antwort auf kurze Fragen | Plan `06` und `08` (weiche Delegation, Form `ein_satz`) |
| V18 | Neue Nutzernachrichten priorisieren | später |
| V19 | Erzwungene Anschlussfragen streichen | Plan `06` |
| V20 | Sprache in Routinen | Plan `12` |
| V21 | Kleine Arbeiten nicht pauschal delegieren | Plan `08` |
| V22 | Persönlicher Kontext für Worker | Plan `05` |
| V23 | Artefakte zwischen Workern | später |
| V24 | Task-Status im Hauptchat zusammenführen | Plan `08` und `09` |
| V25 | Worker-Stillstand nach Zwischenmeldung | Plan `08` |
| V26 | Wiederaufnahme nach Rate-Limit | Plan `08` (Retry mit gespeichertem Plan) |
| V27 | Koordination mehrerer Worker | später |
| V28 | Erfolg unabhängig prüfen | teilweise `08` (Router prüft Zustellung und Ergebnis, bevor er "erledigt" meldet) |
| V29 | Unklare externe Aktionen abgleichen | Plan `10` (unbekannter Ausgang wird geprüft, nie blind wiederholt) |
| V30 | Tool-Ergebnisse bis zur Übernahme sichern | später |
| V31 | Dynamische Modellwahl | später |
| V32 | Günstige Entscheidungen günstig routen | Plan `14` |
| V33 | Strukturierte Ausgaben robuster | teilweise `03` und `14` (Crew- und Adapter-Ausgaben werden geprüft und neu angefragt) |
| V34 | Erfolgreiche Abläufe wiederverwenden | teilweise `04` (Crew schreibt Abläufe als Themenseite; Miniskills bleiben) |
| V35 | Loops von langen Skripten isolieren | später |
| V36 | Ereignisspitzen in Loops | später |
| V37 | Ausgefallene Loops selbst reparieren | teilweise `07` (V39-Wächter für Quellen-Jobs) |
| V38 | Verbrauch je Task dauerhaft | teilweise `15` (Kosten pro Komponente) |
| V39 | Proaktivität ohne Home-Besuch, hängende Jobs | Plan `07` |
| V40 | Mehr Ereignistrigger | teilweise `07` (Gmail-Trigger bleibt, Kalender-Polling bleibt) |
| V41 | Dringender Altbestand bei Erstverbindung | Plan `07` |
| V42 | Aufholen nach Pause | Plan `07` |
| V43 | Fristen neu bewerten ohne Inhaltsänderung | Plan `07` |
| V44 | Offene Verpflichtungen dauerhaft | Plan `09` (Zusagen werden Todos) |
| V45 | Erledigte Vorgänge aus aktivem Tracking | Plan `07` (mit den Budgets wird das Tracking umgebaut) |
| V46 | Dringlichkeit vor der Auswahl der 24 | Plan `07` (harte Regeln und Gate pro Item) |
| V47 | Proaktive Kapazität | entfällt (Budgets weg) |
| V48 | Langzeitkontext in der Triage | Plan `07` (Gedächtnis-Abruf im Weck-Gate) |
| V49 | Unklare Bewertungen | Plan `07` (Fallback `hintergrund`, Router entscheidet) |
| V50 | Ein Fehler bringt Gruppe zum Schweigen | Plan `07` |
| V51 | Quelle vor Meldung neu prüfen | Plan `07` |
| V52 | Benachrichtigungsbudget | entfällt (Budgets weg) |
| V53 | Still protokollieren, informieren, unterbrechen | Plan `07` (`still`, `hintergrund`, `melden`) |
| V54 | Routinen personalisieren | Plan `07` (Gedächtnis und Board in den Briefing-Input) |
| V55 | Private Themen nicht ausblenden | Plan `07` |
| V56 | Briefing mit vollständigerem Überblick | Plan `07` und `09` |
| V57 | Mehr als erste Ergebnisseite | später |
| V58 | Details nachladen | teilweise: Router liest volle Threads per Tool, wenn nötig |
| V59 | Gmail-Filter nach Relevanz | Plan `07` |
| V60 | Ausgehende Mails als offene Vorgänge | später |
| V61 | Kalenderfristen, längere Zeiträume | später |
| V62 | Slack tiefer | später |
| V63 | Notion | später |
| V64 | Drive jenseits neuer Dateien | später |
| V65 | Drive-Kommentare | später |
| V66 | Quellen einheitlich | später |
| V67 | Proaktivität auf allen Kanälen | Plan `13` |
| V68 | Bevorzugter Kanal statt Rundversand | Plan `13` |
| V69 | Zustellstatus und Budget koppeln | entfällt teilweise (Budget weg); Zustellstatus `13` |
| V70 | Notebook privat und steuerbar | später |
| V71 | Entscheidungen nachvollziehbar | Plan `15` (Dashboard, Debug-Ansicht) |
| V72 | Datenschutz auf Nebenpfaden | Plan `14` (Entscheidungs-Router als einziger Weg, gleiche Regeln) |
| V73 | Reale Integrationstests | teilweise `15` (aufgezeichnete Antworten, Evals) |
| V74 | Langfristige Qualität messen | Plan `15` |
| V75 | Self-Hosting verschlanken | teilweise `02` |
| V76 | Backups automatisieren | Plan `02` |
