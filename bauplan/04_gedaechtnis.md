# 04 Gedächtnis

Ziel: Der Assistent weiß alles Relevante, wie ein echter Assistent. Er merkt sich von selbst, korrigiert sich über Zeit, kennt die Herkunft jedes Fakts, versteht Zeit, findet Zusammenhänge und spielt das Richtige automatisch ein. Der Nutzer pflegt nie etwas von Hand (E16 bis E26).

---

## Ist-Zustand (Stand `ba4f55d`)

- `/memory`-Dateien: `semantic/{user,agent,people}.md`, `environments/`, `index.md`, `episodes/YYYY-MM-DD.md` (nur heute, nur Anhängen), `scoped/` (`salix_agent/tool_policy.ex:217-221`, `tools/memory.ex:67-71`).
- Werden nie automatisch geladen (`tool_policy.ex:215`, `salix_web/.../bindings/meeting_memory.ex:6`).
- `memory.write` nur per Tool, nur bei ausdrücklichem "merk dir" oder Vorlieben (`tools/memory.ex:179-181`). Kein Extraktions-Job. Ausnahmen: Meeting-Zusammenfassungen, Miniskills.
- Memory-Tools nur für den Router (`memory.ex:172-186`). Worker können das Gedächtnis nicht lesen (`tool_policy.ex:215,235`).
- Suche nur wörtlich: LIKE im Hot-Verlauf (`salix_agent/session_history/hot.ex:98`), `position()` in ClickHouse (`salix_analytics/session_history_index.ex:23`), Teilstring pro Zeile in `memory.search` (`memory.ex:32-37`). Keine Embeddings in Produktion.
- `history.search/list/get` (`tools/history.ex:43-51`) nur in der aktuellen Session. `internal.search_conversations` und `internal.read_conversation`: Teilstring über die 200 neuesten Unterhaltungen (`salix_im/conversations.ex:27,145`).
- `memory.ask_worker` fragt bis zu 10 frühere Worker-Sessions, standardmäßig aus (`memory_consultation.ex:13`).
- Append-Race V01 im Tagebuch (`memory.ex:39-43,445-465`, `agent_workspace.ex:78-94`).
- Tagebuch-Datum nach Server-Zeit (V04).
- Andockstelle für eingespieltes Wissen existiert: `ProjectKnowledgeContext` (`salix_agent/round.ex:1013`, 250 ms Budget), heute nur für Bridge-for-Teams verdrahtet (`systems/config/config.exs:522`). Bridge-for-Teams hat schon ablösende Aussagen mit `supersedes_id` und `observed_at` (Migration `bridge_for_teams_core/priv/repo/migrations/20260821000002_create_project_knowledge.exs`).
- Originale aller Nachrichten bleiben dauerhaft: Hot-Verlauf, ClickHouse-Index, unveränderliche S3-Segmente (`docs/storage-search.md:44-79`). Darauf baut die Herkunft.

---

## 1. Architektur

| Baustein | Wo | Rolle |
|---|---|---|
| Originale | Comma (bestehend) | Jede Nachricht mit stabiler ID. Quelle der Wahrheit. |
| Gedächtnisdienst | Hindsight-Fork als eigener Container `memory` | Fakten, Entitäten, Links, Themenseiten, Suche |
| Comma-Client | Neues Elixir-Modul `SalixAgent.MemoryService` | HTTP-Client, Zuordnung von Nachrichten-IDs, Tools |
| Erfasser | Comma, Oban-Job nach jedem Turn | Schickt neue Nachrichten an den Dienst zur Faktenextraktion |
| Crew-Gedächtnis-Schreiber | Rebuild (`03`) | Zweite Chance für übersehene Fakten, Themenseiten, Tagesseite |
| Nächtliche Pflege | Comma, Oban-Cron 03:30 Nutzerzeit | Ablauf von Status-Infos, Themenseiten, Entitäten zusammenführen, Kernblock-Kandidaten |
| Abruf | Hidden Helper (`05`) und Tools | Vor jedem Turn und auf Anfrage |
| Kernblock | Router-Kontext, fester Teil | Die wichtigsten Fakten über den Nutzer, immer im Kontext (`03`) |

Entscheidung: Hindsight läuft als geforkter Dienst, nicht nach Elixir portiert. Begründung: Die Suchpipeline (vier Suchwege, Fusion, Reranker, Entitätsauflösung, Links) ist fertig, getestet und aktiv gepflegt (MIT, Postgres). Comma betreibt ohnehin mehrere Dienste (Postgres, Redis, MinIO, ClickHouse). Ein Container mehr ist normal. Der Fork liegt als Git-Subtree unter `services/hindsight/` im Comma-Fork, damit ein Repo alles enthält und `git subtree pull` Upstream-Änderungen holt.

Graphiti wird nicht betrieben (braucht Neo4j, FalkorDB, Kuzu oder Neptune). Übernommen werden nur das Gültigkeitsmodell und die Prompt-Logik für Widersprüche (Apache-2.0, Attribution im Fork beibehalten).

Domänenkonzept: Prüfe `docs/architecture/DOMAIN_CONCEPTS.md`. Bridge-for-Teams hat "Project knowledge assertion" mit `supersedes_id` und `observed_at`. Die persönlichen Fakten sind dieselbe Idee in anderem Scope. Wenn das Konzept auf persönlichen Scope erweiterbar ist, erweitere es. Sonst neuer Eintrag "Personal memory fact" (Owner: Group des Nutzers, Speicher: Gedächtnisdienst, Lebenszyklus unten, Beziehungen zu Message, Task, Board-Item). Der Eintrag muss die Gründe aus diesem Abschnitt nennen.

---

## 2. Datenmodell der Fakten

Ein Fakt ist eine eigenständige Aussage in einem Satz, auf Deutsch, mit absoluten Zeitangaben ("am 15.10.2026", nie "morgen").

| Feld | Bedeutung |
|---|---|
| `id` | Stabile ID |
| `text` | Die Aussage |
| `kategorie` | `person`, `vorliebe`, `entscheidung`, `zusage`, `termin`, `status`, `fakt`, `ziel`, `projekt`, `ablauf`, `kommunikation` |
| `lebensdauer` | `dauerhaft` oder `status` |
| `observed_at` | Wann der Assistent es erfahren hat (Zeitstempel der Quellnachricht) |
| `valid_from` | Ab wann es gilt (Standard: `observed_at`) |
| `valid_to` | Bis wann es gilt. Leer heißt offen. |
| `occurred_at` | Bei Ereignissen und Terminen: wann es passiert |
| `expires_at` | Nur bei `status`: wann es automatisch verfällt |
| `superseded_by` | ID des Fakts, der ihn ersetzt hat |
| `supersedes` | ID des Fakts, den er ersetzt |
| `quelle_art` | `nutzer_gesagt`, `assistent_abgeleitet`, `quelle_extern` (Mail, Kalender, Datei), `import` |
| `source_message_ids` | Liste der Comma-Nachrichten-IDs, aus denen der Fakt stammt |
| `source_external` | Bei externen Quellen: Art und ID (z.B. Gmail-Message-ID) |
| `wichtigkeit` | 0 bis 1 |
| `konfidenz` | 0 bis 1 |
| `entitaeten` | Verknüpfte Entitäten (Personen, Firmen, Orte, Projekte) |
| `status_flag` | `aktiv`, `ersetzt`, `erledigt` (bei Zusagen), `vergangen` (bei Terminen) |

Kategorien im Detail:
- `person`: Wer jemand ist, Beziehung zum Nutzer, Kontaktdaten, Rolle.
- `vorliebe`: Wie der Nutzer etwas mag ("Calls nur nachmittags").
- `entscheidung`: Was entschieden wurde und warum.
- `zusage`: Wer wem was bis wann zugesagt hat (Nutzer oder Assistent). Wird auf dem Board zum Todo (`09`).
- `termin`: Ereignis mit Zeit.
- `status`: Vorübergehender Zustand ("Paket unterwegs", "ist krank", "wartet auf Rückruf").
- `fakt`: Stabiles Wissen ("Steuerberater ist Frau X", "Firma läuft als Einzelunternehmen").
- `ziel`, `projekt`: Langfristige Ziele und laufende Projekte (F12).
- `ablauf`: Wie etwas gemacht wird ("Rechnungen gehen immer an buchhaltung@...").
- `kommunikation`: Wie der Nutzer Antworten will (`06`).

Links zwischen Fakten und Entitäten kommen aus Hindsight: Entität, Zeit, Semantik, Ursache (`caused_by`), dazu Mitvorkommen von Entitäten.

---

## 3. Gültigkeit über Zeit

Regel: Nie zwei widersprüchliche Fakten ohne erkennbare Reihenfolge (E21). Aber nicht pauschal alles aufheben (E22).

Beim Speichern jedes neuen Fakts:
1. Kandidaten finden: Fakten mit denselben Entitäten und ähnlichem Inhalt (Vektor plus Entität, Top 10).
2. Ein Modell-Call (DeepSeek V4.1 Flash, Prompt nach Graphitis Kantenauflösung und Invalidierung) klassifiziert pro Kandidat: `duplikat`, `widerspruch`, `aktualisierung`, `ergänzung`, `unabhängig`.
3. Code wendet an, nie das Modell direkt:
   - `duplikat`: neuer Fakt wird nicht angelegt, nur `source_message_ids` und `observed_at` des alten ergänzt.
   - `widerspruch` oder `aktualisierung`:
     - Alter Fakt hat Lebensdauer `status`: wird gelöscht (Inhalt weg, nur ein Grabstein mit ID, Löschzeitpunkt, Grund bleibt für Verweise).
     - Sonst: alter Fakt bekommt `valid_to` = `valid_from` des neuen, `superseded_by` = neuer, `status_flag` = `ersetzt`. Er bleibt suchbar, wird aber nur gezeigt, wenn die Frage die Vergangenheit betrifft ("wie war das früher?").
   - `ergänzung`: beide bleiben, Link zwischen ihnen.
4. Ist das Modell unsicher (Konfidenz unter 0,6), werden beide Fakten behalten und mit einem Link `moeglicher_widerspruch` verbunden. Die nächtliche Pflege prüft solche Paare mit einem stärkeren Modell (Hauptmodell, Batch).

Ablauf von Status-Infos:
- Beim Erfassen setzt der Extraktor `expires_at`, wenn der Text es nahelegt ("kommt Freitag" ergibt Freitag plus 2 Tage). Sonst Standard 14 Tage.
- Die nächtliche Pflege löscht abgelaufene Status-Fakten.
- Termine werden nach `occurred_at` zu `vergangen`, nicht gelöscht.
- Zusagen werden `erledigt`, wenn das zugehörige Board-Item erledigt ist oder der Erfasser es aus dem Chat erkennt.

"Vergiss das" (F29): Der Router ruft `memory.forget` mit dem Fakt. Der Fakt wird gelöscht (Grabstein). Einspiel-Spuren in künftigen Rebuilds entfallen. Das Archiv der Originalnachrichten bleibt wie heute, Redaktion dort über den bestehenden Weg.

---

## 4. Zeit im Gedächtnis

Zeit ist ein eigenes Konzept (E23, E24).
- Jeder Fakt hat `observed_at`, `valid_from`, optional `occurred_at` und `valid_to`.
- Relative Angaben werden beim Erfassen gegen den Zeitstempel der Quellnachricht in der Zeitzone des Nutzers aufgelöst. Die Auflösung rechnet Code, nicht das Modell: Das Modell liefert die relative Angabe strukturiert (z.B. "nächster Donnerstag, 10:00"), ein Datums-Parser in Code rechnet das absolute Datum. Grund: Entscheidungs- und kleine Modelle sind bei Daten schwach.
- Beim Einspielen steht bei jedem Fakt das absolute Datum und das Alter ("seit 12.03.2026, vor 7 Monaten", "gestern").
- Tagesseiten: Für jeden Tag mit Aktivität schreibt die nächtliche Pflege eine kurze Tagesseite ("Was war am 08.10.2026"): Termine, Entscheidungen, Erledigtes, Offenes. Die Seiten sind Themenseiten vom Typ `tag` und machen Fragen wie "was war letzte Woche" leicht.
- Zeitfragen ("letzte Woche", "im März") übersetzt Code in einen Zeitraum und nutzt die zeitliche Suche des Dienstes.

---

## 5. Suche und Ranking

Pipeline pro Abfrage:
1. Anfrage bauen: aktuelle Nutzernachricht, die letzten 3 Nachrichten, offene Board-Items mit Bezug, Entitäten aus der Nachricht.
2. Hindsight-Recall: vier Suchwege parallel (Bedeutung, BM25, Graph über Entitäten und Links, Zeit), Fusion per Reciprocal Rank Fusion.
3. Kandidaten (Top 50) zusätzlich: Themenseiten und Tagesseiten.
4. Reranker: Qwen3-Reranker bewertet Anfrage gegen Kandidat.
5. Filter: nur `aktiv` und gültig zum aktuellen Zeitpunkt, außer die Anfrage ist historisch (Erkennung per Regel: "früher", "damals", "vorher", "wie war", Zeitraum in der Vergangenheit).
6. Zeitfaktor (Code):
   - Alle Fakten mit `observed_at` in den letzten 48 Stunden: Score-Bonus und niedrigere Einspielschwelle (`05`), dazu wird ihre Quellstelle mit 2 Nachrichten Umgebung mitgeliefert ("großzügiger").
   - Kategorien `status`, `termin`, `projekt`: sanfter Abfall mit Halbwertszeit 30 Tage ab `observed_at` bzw. Abstand zu `occurred_at`.
   - Kategorien `person`, `vorliebe`, `entscheidung`, `fakt`, `ablauf`, `kommunikation`, `ziel`: kein Abfall. Alte stabile Fakten werden nicht schlechter gemacht (E24).
   - Termine in den nächsten 7 Tagen: Bonus.
7. Ergebnis: geordnete Liste mit Score, Begründung (welcher Suchweg), Herkunft.

Fehler (V03): Das Ergebnis unterscheidet "keine Treffer" von "Suche unvollständig oder fehlgeschlagen". Bei Ausfall des Dienstes läuft der Turn ohne Einspielen weiter, der Router erfährt es im Reminder, das Dashboard alarmiert (`15`).

---

## 6. Erfassung

### Nach jedem Turn (Hauptweg)
- Oban-Job nach Abschluss jedes Router-Turns, nicht blockierend.
- Eingabe an den Dienst (Hindsight `retain`): die neuen Nachrichten des Turns (Nutzer, Assistent, Ereignisse, Kurzform relevanter Tool-Ergebnisse), jede mit ihrer Comma-Nachrichten-ID, Zeitstempel, Sprecher.
- Der Fork sorgt dafür, dass jeder extrahierte Fakt die IDs der Nachrichten trägt, aus denen er stammt (Abschnitt 8).
- Danach die Gültigkeitsprüfung aus Abschnitt 3.
- Extraktions-Modell: DeepSeek V4.1 Flash (Together). Extraktions-Prompt auf Deutsch ausgerichtet, mit Kategorien aus Abschnitt 2 und der Pflicht zu absoluten Daten.

### Weitere Quellen
- Worker-Ergebnisse: Wenn eine Task endet, wird ihr Ergebnisbericht (nicht der ganze Verlauf) erfasst, mit Quelle Task-ID und Nachrichten-ID des Berichts.
- Proaktive Ereignisse, auf die der Router reagiert hat (Mail, Termin): Kernfakten daraus mit `quelle_extern`.
- Meeting-Zusammenfassungen (bestehend, `meeting_memory.ex:12`).
- Ausdrückliches "merk dir": `memory.write` bleibt, schreibt sofort mit hoher Wichtigkeit.

### Beim Rebuild (zweite Chance)
Der Crew-Gedächtnis-Schreiber (`03`) liest den ganzen Abschnitt, der verdichtet wird, und liefert fehlende oder falsch erfasste Fakten. Code gleicht gegen den Dienst ab (Duplikate entfallen). Die Verifier prüfen, dass nichts Wichtiges verloren geht.

### Einmal-Import
Ein Migrations-Job liest die bestehenden `/memory`-Dateien (semantic, environments, scoped, episodes, meetings) und speist sie mit `quelle_art = import` ein. Danach sind die Dateien nicht mehr maßgeblich. `memory.read`/`memory.search` lesen aus dem Dienst. Das Tagebuch `episodes/` wird durch Tagesseiten ersetzt. Solange es noch geschrieben wird: Append atomar machen (V01, Compare-and-set auf die Dateiversion in `agent_workspace.ex`).

---

## 7. Themenseiten und Kernblock

### Themenseiten (E42)
- Typen: `person`, `projekt`, `thema`, `tag`, `ablauf`.
- Inhalt: kurzer, aktueller Überblick plus Liste der verknüpften Fakt-IDs.
- Gespeichert im Dienst als eigener Datensatztyp, eingebettet für die Suche.
- Gepflegt von der Crew beim Rebuild und nachts. Die Crew schlägt Änderungen strukturiert vor, Code wendet sie an.
- Erfolgreiche Abläufe (V34): Wenn ein mehrstufiger Auftrag erfolgreich war, schreibt die Crew eine Themenseite `ablauf` ("Wie Rechnungen an Firma X gestellt werden").

### Kernblock
- Die wichtigsten Fakten über den Nutzer, höchstens 1.500 Tokens: Name, Wohnort, Selbstständigkeit und Tätigkeit, wichtigste Personen, stehende Vorlieben, Kommunikationsprofil, aktive Ziele.
- Steht im festen, gecachten Teil des Router-Kontexts (`03`). Wird nur beim Rebuild aktualisiert, damit der Cache zwischen Rebuilds nicht bricht.
- Auswahl: Crew-Mitglied "Kernblock" schlägt vor, Code begrenzt auf das Budget nach Wichtigkeit.

---

## 8. Hindsight-Fork: Änderungen

Die genauen Dateien und Funktionen stehen im Abschnitt "Fork-Punkte" unten (aus der Code-Recherche). Inhaltlich:
1. **Herkunft pro Nachricht:** `retain` nimmt eine Liste von Nachrichten mit ID, Zeit, Sprecher. Jeder extrahierte Fakt speichert die Liste der Nachrichten-IDs, aus denen er stammt. Die API gibt sie bei `recall` mit zurück.
2. **Gültigkeitsfelder** aus Abschnitt 2 als Spalten, plus Migration.
3. **Widerspruchsprüfung und Anwendung** nach Abschnitt 3 als Schritt nach der Extraktion. Prompt nach Graphiti, auf Deutsch geprüft.
4. **Recall-Filter** auf gültig/aktiv und Parameter `historisch`.
5. **Zeitfaktor** aus Abschnitt 5 im Ranking (oder in Comma nach dem Recall, wenn das sauberer ist; dann liefert der Dienst die Rohscores).
6. **Kategorien und Lebensdauer** im Extraktions-Prompt und Schema.
7. **Themenseiten** als Datensatztyp.
8. **Modelle konfigurierbar** auf Qwen3-Embedding und Qwen3-Reranker über die Anbieter aus `02`, LLM DeepSeek V4.1 Flash über Together.
9. **Löschen mit Grabstein.**

---

## 9. Tools für Router und Worker

| Tool | Wer | Zweck |
|---|---|---|
| `memory.search` | Router, Worker | Hybride Suche aus Abschnitt 5 |
| `memory.get` | Router, Worker | Fakt mit Gültigkeit, Herkunft und Links |
| `memory.source` | Router, Worker | Springt zur Originalnachricht: liefert die Nachricht mit 10 Nachrichten davor und danach aus dem Archiv (wie Hivekeeps `read_message`). Funktioniert auch für Nachrichten vor dem Rebuild. |
| `memory.history` | Router | Gültigkeitskette eines Fakts (früher X, seit wann Y) |
| `memory.write` | Router | Ausdrückliches Merken |
| `memory.forget` | Router | "Vergiss das" |
| `history.search` | Router, Worker | Einheitliche Verlaufssuche über Router-Session, alle früheren Router-Sessions und alle Tasks, hybrid (Wortlaut plus Bedeutung), mit Zeitfilter (V05 bis V08) |

- Die Einschränkung "nur Router" für Lesen fällt (`memory.ex:172-186`, `tool_policy.ex:215,235`). Worker schreiben nicht direkt. Ihre Ergebnisse werden erfasst.
- `memory.ask_worker` bleibt aus. Die einheitliche Verlaufssuche ersetzt es.
- Für die semantische Verlaufssuche werden Nachrichten (Router und Tasks) zusätzlich eingebettet. Speicher: eine Tabelle mit `pgvector` in Comma-Postgres (Nachrichten-ID, Session, Zeit, Vektor), befüllt von einem Oban-Job nach jeder Nachricht. Wortlaut-Suche bleibt über Hot-Verlauf und ClickHouse. Fusion beider Ergebnislisten per Reciprocal Rank Fusion, dann Reranker.

---

## 10. Modelle

- Embedding: Qwen3-Embedding. Größe und Anbieter siehe `02`, Abschnitt "Anbieter und Modelle".
- Reranker: Qwen3-Reranker. Größe und Anbieter siehe `02`.
- Extraktion, Widerspruchsprüfung, Themenseiten: DeepSeek V4.1 Flash (Together).
- Nächtliche Prüfung unsicherer Widersprüche: Hauptmodell im Batch.

---

## 11. Abnahme

- Tests: Herkunft bleibt über Rebuild erhalten; Duplikat erzeugt keinen neuen Fakt; Widerspruch bei Vorliebe ersetzt, bei Status löscht; Ablauf löscht Status; Zeitauflösung relativer Angaben; historische Abfrage findet ersetzte Fakten; Worker können lesen, nicht schreiben; Dienstausfall blockiert keinen Turn.
- Eval-Suite "Gedächtnis über Zeit" (`15`) grün.
- Abnahmetest 2 und 6 aus `16`.
