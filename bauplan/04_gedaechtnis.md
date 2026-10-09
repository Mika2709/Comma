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
| Nächtliche Pflege | Comma, Oban-Cron 03:00 Nutzerzeit (Reihenfolge der Nachtjobs: `02` Abschnitt 5) | Ablauf von Status-Infos, Themenseiten, Entitäten zusammenführen, Kernblock-Kandidaten |
| Abruf | Hidden Helper (`05`) und Tools | Vor jedem Turn und auf Anfrage |
| Kernblock | Router-Kontext, fester Teil | Die wichtigsten Fakten über den Nutzer, immer im Kontext (`03`) |

Entscheidung: Hindsight läuft als geforkter Dienst, nicht nach Elixir portiert. Begründung: Die Suchpipeline (vier Suchwege, Fusion, Reranker, Entitätsauflösung, Links) ist fertig, getestet und aktiv gepflegt (MIT, Postgres). Comma betreibt ohnehin mehrere Dienste (Postgres, Redis, MinIO, ClickHouse). Ein Container mehr ist normal. Der Fork liegt als Git-Subtree unter `services/hindsight/` im Comma-Fork, damit ein Repo alles enthält und `git subtree pull` Upstream-Änderungen holt.

Graphiti wird nicht betrieben (braucht Neo4j, FalkorDB, Kuzu oder Neptune). Übernommen werden nur das Gültigkeitsmodell und die Prompt-Logik für Widersprüche (Apache-2.0, Attribution im Fork beibehalten).

Domänenkonzept (entschieden): neuer Eintrag "Personal memory fact" in `docs/architecture/DOMAIN_CONCEPTS.md`. Warum Wiederverwendung nicht reicht: "Project Knowledge / Assertion / Alias" (`DOMAIN_CONCEPTS.md:1137`) steht unter den BFT-Produktkonzepten (Abschnitt 9), ist geteiltes Projektwissen eines Bridge-for-Teams-Projekts und lebt in einer anderen App. Persönliche Fakten gehören genau einem Nutzer, haben eigene Gültigkeitsregeln (Kategorie, Lebensdauer, Ablauf) und einen eigenen Speicher. Eintrag: Identität = Fakt-ID des Gedächtnisdienstes; Scope und Owner = Group des Nutzers; autoritativer Speicher = Gedächtnisdienst (Hindsight-Fork); Lebenszyklus = Abschnitt 3 (aktiv, ersetzt, abgelaufen, gelöscht); Beziehungen = Herkunft auf Message-IDs, optional Task und Board-Item; Code-Links = Comma-Client und `services/hindsight`. Im selben PR wie P2 (`16`).

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

## 5. Suche und Ranking (V02)

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

## 8. Hindsight-Fork: Betrieb und Änderungen

Basis: `vectorize-io/hindsight` Commit `1152717` (2026-10-08), Paket `hindsight-api-slim` 0.10.2, MIT. Pfade unten relativ zu `hindsight-api-slim/hindsight_api/`. Python-Dienst (FastAPI plus eingebetteter Worker), Port 8888, Alembic-Migrationen beim Start.

### Betrieb
- Als Git-Subtree unter `services/hindsight/` im Comma-Fork. Eigenes Image aus `docker/standalone/Dockerfile`, Target `api-only`, `INCLUDE_LOCAL_MODELS=false` (keine lokalen Modelle, alles über APIs).
- Neuer Compose-Dienst `memory`, nur im internen Docker-Netz erreichbar.
- Datenbank: eigene DB `hindsight` im bestehenden Postgres. Das Postgres-Image wird von `postgres:16-bookworm` auf `pgvector/pgvector:pg16` umgestellt (gleiches Datenformat, Volume bleibt). Extensions `vector`, `pg_trgm`, `btree_gin`. `configure` legt DB und Rolle an.
- Auth: `HINDSIGHT_API_TENANT_EXTENSION=hindsight_api.extensions.builtin.tenant:ApiKeyTenantExtension`, Key in `secrets.json`. Ohne das wäre die API offen.
- Eine Bank pro Nutzer: `bank_id = user-<user_id>`.
- BM25 auf Deutsch: `HINDSIGHT_API_TEXT_SEARCH_EXTENSION_NATIVE_LANGUAGE=german`.
- Retain asynchron mit `operation_id` (UUID, idempotent), Status über `GET …/operations/{id}`.
- Reflect wird nicht benutzt (teuer, agentische Schleife). Recall liefert die Fakten.
- Observations (Hindsights eigene Konsolidierung zu "aktuellem Stand") bleiben an. Sie werden beim Recall mit abgefragt.

### API, die Comma nutzt
| Zweck | Aufruf |
|---|---|
| Erfassen | `POST /v1/default/banks/{bank_id}/memories` (`api/http.py:10010`) |
| Suchen | `POST …/memories/recall` (`:6131`) |
| Lesen, invalidieren | `GET/PATCH …/memories/{id}` (`:6012`, `:6044`) |
| Listen mit Filtern | `GET …/memories/list` (`:5781`) |
| Konsolidierung anstoßen | `POST …/consolidate` (`:9663`) |
| Entitäten | `GET …/entities` (`:6709`) |

### Fork-Änderungen
1. **Herkunft pro Fakt.** Comma sendet pro Erfassung ein Item, dessen `content` ein JSON-Array von Turns ist (`{id, role, time, content}`), mit `timestamp` der letzten Nachricht und `document_id = turn:<turn_id>` (eindeutig, sonst ersetzt `update_mode=replace` alte Fakten, `api/http.py:1213-1218`). Chunking schneidet an Turn-Grenzen (`engine/retain/fact_extraction.py:822-825,881`).
   - Neues Feld `source_message_ids: list[str]` in `ExtractedFact` (`fact_extraction.py:261`) und den Varianten (`:368`, `:467`, `:517`), Regel im Prompt `_BASE_FACT_EXTRACTION_PROMPT` (`:1058`): "Nenne die IDs der Turns, aus denen der Fakt stammt." Vorbild: Graphitis `episode_indices`.
   - Parsing und Prüfung gegen die IDs im Chunk (`fact_extraction.py` ab etwa `:2225`), durchreichen über `ProcessedFact` (`engine/retain/types.py:323,357,450-494`), Insert (`engine/memories/pg/writes.py:31-125`, `engine/db/ops_postgresql.py` `insert_facts_batch`).
   - Alembic-Migration: Spalte `source_message_ids text[]` mit GIN-Index in `memory_units` **und** `invalidated_memory_units` (sonst scheitert `invalidate_memory`, `writes.py:359-377,418-450`).
   - Spaltenlisten im Recall: `cols` in `engine/memories/pg/recall.py:115`, `pool_cols` (`:472`), `RetrievalResult` (`engine/search/types.py:47,110`), `RecallResult` (`api/http.py:600`), Export/Import (`engine/transfer/schema.py`).
2. **Kategorie, Lebensdauer, Gültigkeit.** Spalten `kategorie`, `lebensdauer`, `valid_from`, `invalid_at`, `superseded_by uuid`, `expires_at`, `status_flag` in `memory_units` und `invalidated_memory_units`, Index `(bank_id, invalid_at)`.
   - Extraktions-Schema um `kategorie`, `lebensdauer`, `valid_at`, `invalid_at`, `expires_hint`, `relative_expression` erweitern. Das heute verworfene `fact_kind` speichern statt wegwerfen (`fact_extraction.py:2236-2239`).
   - Ersetzte Fakten bleiben mit `invalid_at` in `memory_units` (für "was galt früher?"). Nur Löschen und abgelaufene Status-Fakten wandern ins Archiv `invalidated_memory_units`.
3. **Recall-Filter.** Standard "gültig jetzt": `invalid_at IS NULL OR invalid_at > now` und `expires_at IS NULL OR expires_at > now`. In **jeden** Suchweg einbauen, nach dem Muster `updated_range_clause`: semantic plus BM25 (`engine/memories/pg/recall.py:28`, `engine/sql/postgresql.py:320,351`), temporal (`pg/recall.py` etwa `:470-495`), Graph-Seeds und Expansionen (`engine/memories/pg/link_expansion.py:51,299,373`, SQL in `engine/db/ops_postgresql.py`). `RecallRequest` (`api/http.py:492`) bekommt `include_invalid` und `valid_at`.
4. **Widerspruchsprüfung.** Neuer Worker-Job nach Phase 2 des Retain (Einhängepunkt nach `_insert_facts_and_links`, `engine/retain/orchestrator.py:762`, oder als Operationstyp neben `run_consolidation_job`, `engine/consolidation/consolidator.py:1469`), damit Retain nicht blockiert.
   - Kandidaten: semantische Nachbarn aus Phase 1 (`engine/memories/pg/links.py:581ff`, Ähnlichkeit ≥ 0,7), Fakten mit gemeinsamen Entitäten (`unit_entities`), nur gültige.
   - Prompt nach Graphiti `resolve_edge` (`graphiti_core/prompts/dedupe_edges.py:44-101`): zwei Listen, Ausgabe `duplicate_facts` und `contradicted_facts`, plus unser Ergebnis `ergänzung`. Auf Deutsch getestet. Apache-2.0-Hinweis im Kopf der Datei.
   - Zeitlogik nach `resolve_edge_contradictions` (`graphiti_core/utils/maintenance/edge_operations.py:547-582,853-866`): Ist der alte Fakt älter, bekommt er `invalid_at = valid_from des neuen` und `superseded_by`. Ist der neue älter als ein Widerspruchskandidat, läuft der neue sofort ab.
   - Regeln aus Abschnitt 3: Duplikat ergänzt nur Herkunft, Status-Widerspruch löscht ins Archiv, Unsicherheit unter 0,6 erzeugt einen Link `moeglicher_widerspruch`.
   - Danach abgeleitete Observations bereinigen (`fact_storage.delete_stale_observations_for_memories`, `engine/retain/fact_storage.py:147`) und `audit_log`-Eintrag.
5. **Status-Ablauf.** Neue Aufgabe in `MaintenanceLoop` (`engine/maintenance.py:114`, Muster `_run_retention` `:249-258` und `_purge_table_in_batches` `:295`): abgelaufene Status-Fakten in Batches mit `FOR UPDATE SKIP LOCKED` ins Archiv verschieben (`invalidate_memory`). Konfiguration `HINDSIGHT_API_STATUS_SWEEP_INTERVAL_SECONDS` (Standard 3600).
6. **Zeitgewichtung.**
   - `_RECENCY_ALPHA` (`engine/search/reranking.py:35`, heute fest 0,2, also nur ±10 Prozent) konfigurierbar machen, Standard 0,8.
   - Neue Kurve `fresh_boost` in `compute_recency_decay` (`reranking.py:56-78`) und `RECENCY_DECAY_FUNCTIONS` (`config.py:1350`): Alter bis 48 h = 1,0, danach exponentiell mit Halbwertszeit 30 Tage.
   - Recency nach Aussagezeit: In `_recency_for_unit` (`reranking.py:163`) `mentioned_at` vorrangig statt `occurred_start`.
   - Kategorie-Regel: Für `person`, `vorliebe`, `entscheidung`, `fakt`, `ablauf`, `kommunikation`, `ziel` ist der Recency-Faktor neutral (1,0), außer dem Frische-Bonus der ersten 48 h. Abfall nur für `status`, `termin`, `projekt`.
   - Frische schützen: In `trim_merged_candidates` (`engine/search/recall_boost.py:195`) Plätze für Fakten mit `mentioned_at ≥ now - 48h` reservieren.
   - Eigener Suchweg "frisch": SQL-Arm nach `mentioned_at DESC` mit niedriger Ähnlichkeitsschwelle, fließt in die RRF (`engine/memory_engine.py:9800-9806`, Store `engine/memories/postgres.py:124`).
   - Comma setzt bei jedem Recall `query_timestamp` auf jetzt.
7. **Deutsche Zeitangaben.** Der Regex-Fallback `_infer_temporal_date` (`fact_extraction.py:89-125`) kennt nur Englisch. Statt ihn zu erweitern: Das neue Feld `relative_expression` wird nach der Extraktion mit `dateparser` (ist schon Abhängigkeit) in Sprache de und en aufgelöst, Basis `mentioned_at` in der Zeitzone des Nutzers. Gelingt das, überschreibt es `occurred_start`. Weicht es vom Modell-Datum ab, wird beides protokolliert.
8. **Themenseiten.** Hindsight hat Mental Models (gepflegte Zusammenfassungen, Tabelle `mental_models`, Refresh über Reflect). Prüfe die API dafür. Wenn Mental Models programmatisch angelegt und mit vorgegebenem Text aktualisiert werden können, sind sie die Themenseiten. Sonst wird im Fork ein Datensatztyp `topic_page` (Titel, Typ, Text, verknüpfte Fakt-IDs, Embedding) mit CRUD-Routen ergänzt. In beiden Fällen schreibt die Crew den Text, nicht Reflect.
9. **Upstream:** Alle Änderungen als eigene Module und Migrationen, damit `git subtree pull` von `vectorize-io/hindsight` möglich bleibt. Tests im Fork für jede Änderung.

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

Das Embedding-Modell muss vor den ersten echten Daten feststehen. Hindsight bricht ab, wenn die Dimension bei gefüllter Tabelle wechselt (`migrations.py:763`), und pgvector-HNSW erlaubt höchstens 2000 Dimensionen (`migrations.py:732`).

| Zweck | Modell | Anbieter und Weg |
|---|---|---|
| Embedding | Qwen3-Embedding-8B mit 1536 Dimensionen (Matryoshka-Kürzung) | DeepInfra, OpenAI-kompatibel `https://api.deepinfra.com/v1/openai/embeddings`, Hindsight `EMBEDDINGS_PROVIDER=openai`, `EMBEDDINGS_OPENAI_BASE_URL=https://api.deepinfra.com/v1/openai`, `EMBEDDINGS_OPENAI_MODEL=Qwen/Qwen3-Embedding-8B`, `EMBEDDINGS_OPENAI_DIMENSIONS=1536` |
| Reranker | Qwen3-Reranker-4B | DeepInfra über Hindsights `litellm-sdk`-Reranker (Modell `deepinfra/Qwen/Qwen3-Reranker-4B`), `RERANKER_MAX_CANDIDATES=50` |
| Extraktion, Widerspruch, Observations | DeepSeek V4.1 Flash | Together, `HINDSIGHT_API_LLM_PROVIDER=openai`, `LLM_BASE_URL=https://api.together.xyz/v1`, `LLM_MODEL=deepseek-ai/DeepSeek-V4.1-Flash` |
| Prüfung unsicherer Widersprüche (nachts) | Hauptmodell | über Comma |

Regeln beim Einrichten (vor den ersten Daten, mit echten Calls):
1. Prüfe, ob DeepInfra `dimensions` beachtet (Vektorlänge 1536). Wenn nicht: Qwen3-Embedding-0.6B (1024 Dimensionen, passt ohne Kürzung).
2. Query-Präfix setzen, weil der Server kein Instruct-Template anwendet: `HINDSIGHT_API_EMBEDDINGS_QUERY_PREFIX` = "Instruct: Given a message from a conversation with a personal assistant, retrieve stored facts about the user that help to answer it\nQuery: ". Passage-Präfix leer.
3. Prüfe den Reranker-Weg über `litellm-sdk`. Wenn er nicht geht: Hindsights eingebauter Provider `siliconflow` mit `Qwen/Qwen3-Reranker-8B` (dann zusätzlich SiliconFlow-Key beim Nutzer anfragen).
4. Miss die Reranker-Latenz für 50 Kandidaten. Liegt p95 mit Qwen3-Reranker-8B unter 700 ms, nimm 8B, sonst bleibt 4B.
5. Prüfe, ob Together für DeepSeek `response_format` (JSON-Modus) beachtet, und setze `HINDSIGHT_API_LLM_OPENAI_COMPATIBLE_JSON_MODE` entsprechend.

## 11. Abnahme

- Tests: Herkunft bleibt über Rebuild erhalten; Duplikat erzeugt keinen neuen Fakt; Widerspruch bei Vorliebe ersetzt, bei Status löscht; Ablauf löscht Status; Zeitauflösung relativer Angaben; historische Abfrage findet ersetzte Fakten; Worker können lesen, nicht schreiben; Dienstausfall blockiert keinen Turn.
- Eval-Suite "Gedächtnis über Zeit" (`15`) grün.
- Abnahmetest 2 und 6 aus `16`.
