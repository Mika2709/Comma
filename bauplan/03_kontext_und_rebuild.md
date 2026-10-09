# 03 Endlos-Chat: Kontext und Rebuild

Ziel: Ein einziger Chat für immer. Kein rollendes Fenster (jede Nachricht wäre ein Cache-Miss). Der Kontext wird neu aufgebaut, wenn der Cache sowieso abgelaufen ist und der Kontext groß genug ist. Erledigtes wird verdichtet, Offenes bleibt wörtlich, Wissen wandert ins Gedächtnis. Eine Crew aus günstigen Modellen macht das parallel, zwei Verifier prüfen. Die Summary wächst nicht von Tag zu Tag.

Pfad-Abkürzungen: `VK/` = `systems/native/verified_kernel/runtime/VerifiedKernel/`, `VKLIB/` = `systems/native/verified_kernel/lib/`, `SA/` = `systems/apps/salix_agent/lib/salix_agent/`.

---

## Ist-Zustand (Stand `ba4f55d`)

### Aufbau eines Requests
`Request.dispatch` und `build` (`VK/Session/Request.lean:266-314`):
1. Prompt-Snapshot als erste `summary`-Nachricht (gespeichertes `system_prompt`, `VK/Session/Query/Reply.lean:398-405`).
2. Compaction-Präfix: `<compacted-context>…</compacted-context>` mit `id 0` (`VK/Session/Query/Compaction.lean:152-160`).
3. Live-Nachrichten nach dem Watermark `compacted_through` (`:108-130`), mit Source-Headern und `input_time`.
4. Aktivierungs-Delta (Zeitkontext, Queue-Druck, Rundenbudget), angehängt, bricht keinen Präfix (`SA/context_providers.ex:34-81`).
5. Miniskill-Bodies (nicht im Transkript).
6. Tail: Obligation-Reminder und `turn:`-Zeile, nicht cache-markiert.

Bei Anthropic werden 1 und 2 zum `system`-String (`VK/Provider/Messages.lean:64-71`). Eine neue Summary ändert also den ersten gecachten Block. Das ist beim Rebuild gewollt. Teile 5 und 6 werden pro Request neu gebaut und nicht gespeichert. Das bricht bei Anthropic ab Opus 5.5 die Thinking-Bindung (Abschnitt 7).

Anthropic sendet Thinking früherer Runden mit, solange das Modell gleich ist (`provider_meta.anthropic_thinking`, `VK/Provider/Messages.lean:34-38`).

### Auslöser heute
- Kette: `config_facts/1` (`SA/internal_session_actor.ex:1535-1540`) → Drive `activation_plan` (`VK/Session/Drive.lean:475-485`) → `CompactionHost.activationPlan` (`VK/Session/CompactionHost.lean:520-524`) → `required` (`:227-232`) → `StateQuery.shouldCompact` (`VK/Session/Query/State.lean:228-245`).
- Regel: beobachtete Prompt-Tokens plus Miniskill-Bytes > 0,9 × Fenster. Faktor 0,9 hart in `triggerLimit` (`:221-225`). Fenster aus `context_tokens`, sonst 128.000 (`CompactionHost.lean:42,141-146`).
- `prepare` (`CompactionHost.lean:251-288`) prüft die Größe ein zweites Mal und lehnt mit `below_prefilter` (`:271`) oder `below_context_window` (`:281`) ab. Eine neue Regel muss in `required` **und** in `prepare` stehen.
- Geprüft wird nur bei Aktivierung (neuer Input, Wake, `reprocess`). Es gibt keinen Zeit-Auslöser.
- `observedPromptTokens` (`State.lean:67-84`) ist nach einer Compaction 0, bis die nächste Antwort kommt.
- Overflow-Recovery: Lehnt der Anbieter wegen Kontextgröße ab, folgt genau eine Recovery-Compaction pro Watermark (`VK/Session/Loop.lean:366-380`, `Drive.lean:355-357,529-533`, `SA/context_overflow.ex:1-9`).
- Byte-Schwelle `compaction_threshold` existiert nur als Test-Seam, Standard `nil` (`State.lean:234-243`, `VKLIB/salix_verified_kernel.ex:1350`).

### Summary heute
- Lean baut den Request mit der Anweisung `CompactionHost.instruction` (`CompactionHost.lean:48-54`): "Summarize the conversation so far. Preserve key facts, decisions, file paths, tool results, and any state needed to continue working. …". Elixir ruft das Hauptmodell (`SA/compaction.ex:449-502`).
- Host-Seam existiert: `opts[:summarizer]` oder `config :salix_agent, :summarizer` (`SA/compaction.ex:450-453,476-477`). Der Kernel übernimmt `{:summary, text}` ungeprüft (`CompactionHost.lean:401`), `{:summary_text, text}` braucht `<compaction-summary>`-Tags (`:402-406`). Signatur `(prev_summary, live_messages)` ohne Agent-, Session- und Billing-Kontext. Läuft nicht bei Strategie `openai_responses`.
- Wörtlich bleibt nur, was nach der frühesten offenen Grenze liegt (`dropUnfinished`, `CompactionHost.lean:175-186`, `Query/Compaction.lean:168-230`), plus dauerhafter Zustand außerhalb des Transkripts (Antwortpflichten, `wait`, Async-Tools, Queue). Eine strukturierte Liste offener Punkte gibt es nicht.
- Kein Verifier.
- Commit mit Fence (`CompactionHost.lean:461-490`), Archiv in S3-Segmente (`SA/compaction.ex:269-305`, `SA/internal_session_store.ex:1986-2020`).
- Redaction-Overlay `session_microcompact` ersetzt Inhalte von Nachrichten nur lesend (`VK/Session/History.lean:26-52`), ein Text pro Event, `provider_meta` (Thinking) bleibt unberührt.

### Cache heute
- Anthropic: `cache_control: {type: "ephemeral"}` ohne `ttl`, also 5 Minuten (`VK/Provider/Request.lean:26`). System-Block plus bis zu 3 Nachrichten-Breakpoints, Tail nicht markiert.
- OpenAI: `prompt_cache_key` pro Session (`SA/llm.ex:820-870`), keine Retention-Einstellung.
- Kein Feld `cache_ttl_seconds` irgendwo. Kein Feld für den Zeitpunkt des letzten Provider-Calls. Nächster Wert: `created_at` der neuesten Assistant-Nachricht (`VK/Session/LoopHost.lean:289`). `execution_timing` darf laut Vertrag nicht für Fristen genutzt werden (`SA/execution_timing.ex:2-5`).
- Der Session-Actor hat keinen Idle-Timer (`SA/internal_session_actor.ex:2544-2552`, `SA/session_residency.ex:3`).

### Beweise und TLA
- Kein Theorem deckt die Compaction-Host-Queries ab (`systems/native/verified_kernel/README.md:140-143`), nur Laufzeittests in `systems/native/verified_kernel/test/compaction_host_test.exs`. Reducer-Theoreme (`WorkFrames`, `WorkCompaction`, `Properties`, `WorkNonRetiring`, `AppendOnly`) betreffen den Commit, nicht den Auslöser.
- Kein TLA-Modell hat Compaction zum Gegenstand. `CompactionFence.tla` und `CompactionContinuation.tla`, auf die `SA/compaction.ex:18-21` verweist, existieren nicht mehr (historische Verweise laut `tla/README.md:41-47`). Nicht neu anlegen.

---

## 1. Auslöser neu

### Regeln
1. **Rebuild bei Cache-Ablauf:** Der Kontext wird neu aufgebaut, wenn
   - seit der letzten Cache-Berührung des Hauptmodells mindestens `cache_ttl_seconds` vergangen sind, und
   - die beobachteten (oder geschätzten) Prompt-Tokens mindestens `rebuild_min_tokens` betragen (Standard 100.000), und
   - die Session nicht aktiv ist (bestehende Admission, `Query/Compaction.lean:322-327`).
2. **Notbremse:** Kontext erreicht `window - reserve`. `reserve` = max(8 Prozent des Fensters, 32.000 Tokens) für Ausgabe und Tool-Ergebnisse. Ersetzt den festen Faktor 0,9 in `triggerLimit`.
3. **Overflow-Recovery** bleibt als zweites Netz.
4. **Kein weiterer Größen-Auslöser** (E31). Die Byte-Schwelle bleibt nur Test-Seam.

"Letzte Cache-Berührung" = Maximum aus `created_at` der neuesten Assistant-Nachricht mit `input_tokens > 0` und dem Zeitpunkt des letzten Keep-alive-Calls (Abschnitt 6). Den Keep-alive-Zeitpunkt hält der Session-Actor im Speicher und gibt ihn als Fakt in `Compaction.required_facts` (`SA/compaction.ex:115-122`) an den Kernel. Geht er bei Eviction verloren, gilt der Cache als abgelaufen (konservativ, schlimmstenfalls ein Rebuild zu früh). Kein neues dauerhaftes Session-Feld, kein neuer Reducer, damit die Beweise unberührt bleiben.

Token-Stand (V10): `observedPromptTokens`. Ist er 0 oder unbekannt (direkt nach einem Rebuild oder bei fehlender Usage), schätzt der Kernel konservativ aus `contextBytes / 3`.

### Wann geprüft wird
- **Proaktiv im Leerlauf (Hauptweg):** Nach jeder Antwort des Hauptmodells registriert der Session-Actor einen dauerhaften Timer über `SalixStore.Timers.register/1` (`systems/apps/salix_store/lib/salix_store/timers.ex:1-55`) mit neuem Kind `cache_expiry`, Deadline = Antwortzeit + `cache_ttl_seconds`, deterministische `timer_id` pro Session (überschreibt den vorigen). `SalixCluster.Timers` (`systems/apps/salix_cluster/lib/salix_cluster/timers.ex:266-283`) bekommt eine eigene `deliver_timer`-Klausel für `cache_expiry`, die die Session nur weckt (`handle_cast(:wake)`, `SA/internal_session_actor.ex:475-477`), keine Nachricht einliefert. Der Kernel prüft bei dieser Aktivierung die Regeln. Liegt nichts an, passiert nichts.
- **Fallback bei Aktivierung:** Kommt eine Nachricht und der Rebuild ist trotz abgelaufenem Cache nicht gelaufen (Timer verloren), läuft er vor der Runde. Der Nutzer wartet dann einmal länger. Die Reaktion 🔍 ist hier nicht passend; stattdessen zeigt der Client den normalen Tipp-Indikator.
- Kardinalität (AGENTS.md): genau ein `cache_expiry`-Timer pro Router-Session. Nur Router-Sessions bekommen ihn (Worker-Sessions enden mit der Task).

### Kollision mit Nutzer-Input
- Läuft ein Rebuild und es kommt eine Nachricht, wartet der Turn bis zu 30 Sekunden auf das Ende des Rebuilds. Danach läuft der Turn mit dem alten Kontext, der Rebuild wird verworfen und beim nächsten Cache-Ablauf neu versucht. Der Commit-Fence (`CompactionHost.lean:461-490`) verhindert inkonsistente Zustände.

### Änderungen im Kernel (Lean)
1. `StateQuery.triggerLimit` (`VK/Session/Query/State.lean:221-225`): Faktor 0,9 ersetzen durch `window - reserve` mit konfigurierbarer Reserve.
2. `CompactionHost.required` (`VK/Session/CompactionHost.lean:227-232`) und `CompactionHost.prepare` (`:251-288`, zwischen Prefilter und `below_context_window`): neue Gründe `cache_expired` und `emergency` neben `overflow_recovery`. `prepare` darf `cache_expired` nicht mit `below_context_window` ablehnen, solange `rebuild_min_tokens` erreicht ist.
3. Neue Query `last_cache_touch_at` nach dem Muster von `observedPromptTokens` (`State.lean:67-84`), liest `created_at` derselben Nachricht, nimmt das Maximum mit dem Fakt aus dem Actor.
4. Uhrzeit per `observe (a "time")` (schon genutzt in `CompactionHost.lean:56-57`).
5. Konfiguration: `rebuild_min_tokens`, `rebuild_reserve_tokens`, `rebuild_keep_recent_tokens`, `rebuild_keep_recent_messages` als `observe {:config, :salix_agent, key, default}` und in `@config_keys` eintragen (`VKLIB/salix_verified_kernel.ex:1345-1351`), damit keine Continuation-Rundreise entsteht. `cache_ttl_seconds` kommt über die Modell-Config (Abschnitt 5).
6. Plan mit **behaltenem Ende** (Abschnitt 2): Der Plan deckt nur Nachrichten bis zu einem Schnitt ab, der die jüngsten `rebuild_keep_recent_tokens` (Standard 25.000) bzw. mindestens `rebuild_keep_recent_messages` (Standard 8) wörtlich lässt. Der Schnitt trennt nie Tool-Call und Tool-Ergebnis und nie eine offene Aktivierung (bestehende Grenzen aus `Query/Compaction.lean:168-230` gelten weiter).
7. Neue Atome in `@protocol_atoms` (`VKLIB/salix_verified_kernel.ex:34-…`).
8. Neue Queries in der `table` registrieren (`VK/Session/Query.lean:29-31`).
9. Tests in `systems/native/verified_kernel/test/compaction_host_test.exs` (ab `:97` Trigger/Admission/Backoff): Cache abgelaufen über und unter Schwelle, Notbremse, Schätzung bei fehlender Usage, behaltenes Ende, Schnitt nie zwischen Tool-Call und Ergebnis, Keep-alive-Fakt verschiebt den Ablauf.
10. `systems/native/verified_kernel/scripts/check.sh` muss grün sein (Beweise, Axiom-Audit, NIF-Build, Elixir-Tests). Da kein neuer Reducer und kein neues Event entsteht, sollten die Reducer-Theoreme unverändert gelten. Die README-Aussage "No theorem covers them" (Host-Queries) bleibt richtig.
11. TLA: unverändert. Im PR begründen: Kein retained Modell deckt Compaction ab, Archiv-Grenze `compacted_seq` (`tla/salix/ArchiveAppend.tla:47-51,80`, `SessionHotArchive.tla:205-206`) bleibt unberührt, weil Commit und Archiv-Pfad gleich bleiben. `make tla` laufen lassen.

---

## 2. Was nach einem Rebuild im Kontext steht

```
[Prompt-Snapshot]                       unverändert
<compacted-context>
  Kernblock                             höchstens 1.500 Tokens (aus dem Gedächtnis, 04)
  Offene Punkte (wörtlich)              ohne Budget, jedes mit Nachrichten-ID
  Laufende Themen                       höchstens 1.500 Tokens
  Verlauf                               höchstens 6.000 Tokens, zeitlich gestaffelt
  Aktionsprotokoll                      höchstens 1.500 Tokens, eine Zeile pro externer Aktion
  Hinweise auf Werkzeuge                fester Text
</compacted-context>
[Live-Nachrichten: behaltenes Ende]     wörtlich, große Tool-Ausgaben gekürzt
```
(Das ist ein Strukturbild, kein Code.)

### Abschnitte
- **Kernblock:** Die wichtigsten Fakten über den Nutzer (`04` Abschnitt 7). Steht vorne, ändert sich nur beim Rebuild.
- **Offene Punkte (wörtlich):** offene Aufgaben und Zusagen, wartende Freigaben, unbeantwortete Fragen des Nutzers oder des Assistenten, laufende Helfer, angekündigte Rückmeldungen. Jeder Punkt mit dem wörtlichen Zitat der Originalnachricht (gekürzt nur, wenn über 600 Zeichen, dann mit Verweis), Datum und Nachrichten-ID. Unfertiges darf so lange und so groß bleiben wie nötig (E30).
- **Laufende Themen:** Was gerade läuft, mit Verweis auf Themenseiten.
- **Verlauf, zeitlich gestaffelt:** Heute und gestern ausführlich, die 5 Tage davor je höchstens 3 Zeilen, alles Ältere fällt aus der Summary. Ältere Details liegen im Gedächtnis (Tagesseiten, Fakten) und im Archiv. Das hält die Summary dauerhaft klein (E34).
- **Aktionsprotokoll:** Eine Zeile pro externer Aktion der letzten 48 Stunden ("08.10. 14:02 Mail an X gesendet, Freigabe f12"). Das ist die "Tool-Ausgabe auf eine Zeile" (E36).
- **Hinweise:** "Ältere Einzelheiten: `memory.search`, `history.search`, `memory.source`."

### Behaltenes Ende
- Die jüngsten Nachrichten bleiben wörtlich (Abschnitt 1, Punkt 6).
- Große Tool-Ausgaben darin (über 2.000 Zeichen) werden über das bestehende Redaction-Overlay (`session_microcompact`, ein Event pro Nachricht mit eigenem Text, `VK/Session/History.lean:26-52`) auf eine Zeile plus Kernergebnis gekürzt. Das Original bleibt im Journal und Archiv. Das passiert nur im Rebuild, weil es den Präfix ändert (V11).
- Thinking im behaltenen Ende wird beim Rebuild komplett entfernt, bei allen Anthropic-Modellen (Abschnitt 7, Schritt 3). Text und Tool-Calls bleiben wörtlich. Grund: Ab Opus 5.5 ist jeder Thinking-Block an den Präfix davor gebunden. Die neue Summary ändert diesen Präfix, das alte Thinking wäre ungültig und ergäbe Fehler 400. Andere Anbieter haben keine Bindung. Ihre Reasoning-Felder (`reasoning_content`, `reasoning_details`, `VK/Provider/Messages.lean:14,91`) bleiben, nur die Blöcke aus der R5-Liste fallen weg (Abschnitt 3).

### Größenwächter
- Ein Code-Schritt misst jeden Abschnitt gegen sein Budget. Überschreitet "Verlauf" sein Budget, verdichtet ein zusätzlicher Crew-Call die ältesten Tage weiter. "Offene Punkte" sind vom Budget ausgenommen.
- Dashboard (`15`) zeigt die Größe jedes Abschnitts pro Rebuild. Alarm, wenn die Summary ohne offene Punkte 7 Rebuilds in Folge wächst.

---

## 3. Cleanup-Crew

### Rollen
Jede Rolle ist ein eigener API-Call mit vollem Kontext des zu verdichtenden Bereichs und einer festen Aufgabe (E38). Alle laufen parallel. Ausgabe immer strukturiert (JSON nach Schema), nie freier Text, der direkt irgendwo landet (E40).

**Normalfall: Jede Rolle bekommt den ganzen zu verdichtenden Bereich** mit Nutzer- und Assistententext, Thinking, Tool-Calls, Tool-Ergebnissen, alter Summary und dauerhaftem Zustand (Antwortpflichten, `wait`, Async-Tools, Board, Kernblock, Themenseiten). Die mittlere Spalte gilt nur, wenn der Bereich nicht ins Fenster des Crew-Modells passt (E41, Absatz unter der Tabelle).

| Rolle | Sicht nur bei Übergröße (E41) | Ausgabe |
|---|---|---|
| R1 Verlaufsschreiber | Text (Nutzer, Assistent) plus alte Summary | Abschnitte "Laufende Themen" und "Verlauf" |
| R2 Offene-Punkte-Sammler | Text plus Tool-Calls plus dauerhafter Zustand | Liste offener Punkte mit Nachrichten-IDs |
| R3 Gedächtnis-Schreiber | Text plus Tool-Ergebnisse | Neue oder korrigierte Fakten mit Herkunft (`04`) |
| R4 Aktions- und Tool-Kürzer | Tool-Calls und Ergebnisse | Aktionsprotokoll plus Kürzungen für große Tool-Ausgaben im behaltenen Ende |
| R5 Begründungs-Auswerter | Text plus Thinking | Entscheidungsbegründungen, die erhalten bleiben müssen (fließen in R1-Ergebnis und als Fakten `entscheidung` ins Gedächtnis), plus Liste der Thinking-Blöcke im behaltenen Ende, deren Inhalt keine offene Überlegung trägt (E41). Bei Anbietern ohne Bindung entfernt Code beim Rebuild genau diese Reasoning-Felder. Bei Anthropic entfernt Code ohnehin alle (Abschnitt 7); R5 überführt dann jede noch nötige Überlegung aus dem behaltenen Ende als Fakt `entscheidung` oder als offenen Punkt, bevor sie verschwindet. |
| R6 Themen- und Tagesseiten-Pfleger | Text plus bestehende Themenseiten | Änderungen an Themenseiten und Tagesseiten (`04`) |
| R7 Kernblock-Pfleger | Text plus aktueller Kernblock plus Top-Fakten | Neuer Kernblock im Budget |
| R8 Board-Abgleicher | Text plus aktuelles Todo-Board (`09`) | Fehlende oder veraltete Board-Items |

- Leichter Überlapp ist gewollt (E37): R2 und R8 sehen beide Zusagen, R3 und R5 beide Entscheidungen. Code dedupliziert.
- Modell aller Rollen: DeepSeek V4.1 Flash bei Together AI (E37). Kontextgröße des Modells: siehe `02`.
- Ist der zu verdichtende Bereich größer als das Kontextfenster des Crew-Modells, bekommt jede Rolle nur ihre Sicht nach Typ (E41): R1 nur Text, R5 Text plus Thinking, R4 nur Tool-Calls. Reicht auch das nicht, wird zeitlich in Blöcke geteilt (älteste zuerst), und jede Rolle trägt eine laufende Notiz von Block zu Block. Das ist der letzte Ausweg.
- Timeout pro Call 90 s, ein Retry mit kaputtem JSON (V33: Schema-Prüfung, bei Verstoß einmal neu anfragen mit Fehlermeldung).

### Anwenden (Code)
- Der Code baut die Summary aus R1, R2, R4, R7 nach dem Schema in Abschnitt 2.
- R3 und R5 gehen an den Gedächtnisdienst (`04`, mit Duplikatprüfung).
- R6 an die Themenseiten.
- R8 an das Board (`09`), als Vorschläge für den Router-Kontext, nicht als direkte Änderung sichtbarer Items. Ausnahme: neue Zusagen werden direkt als Items angelegt.
- R4-Kürzungen als Overlay-Events.
- Thinking im behaltenen Ende: Commit setzt `thinking_cutoff_id` (Abschnitt 7, Schritt 3).
- Reasoning bei Anbietern ohne Bindung: Der Commit speichert die Nachrichten-IDs aus der R5-Liste als `reasoning_drop_ids` (höchstens 500 IDs, ersetzt beim nächsten Rebuild). Der Chat-Encoder (`VK/Provider/Messages.lean:90-91`, `chat_message_extra`) lässt für diese IDs die `reasoningFields` weg. Das Overlay reicht dafür nicht, weil es `provider_meta` nicht erreicht.

---

## 4. Verifier (V12)

Zwei Verifier mit unterschiedlichen Modellen prüfen das Gesamtergebnis (E39):
- **Verifier 1:** DeepSeek V4.1 Flash (Together), eigene Instanz, eigener Prompt.
- **Verifier 2:** Claude Haiku 5.5 über OpenRouter, ID `anthropic/claude-haiku-5.5` (beim Einrichten mit echtem Call prüfen und die datierte Fassung pinnen, falls OpenRouter eine listet). Es muss sich nur von Verifier 1 unterscheiden, nicht vom Hauptmodell. Haiku und DeepSeek sind immer verschiedene Familien, also gibt es keine Ausnahme. Schlägt der Prüf-Call fehl: `openai/gpt-5.5-mini` über OpenRouter, gleiche Regel.

Beide bekommen: den Originalbereich, die neue Summary, die Crew-Ausgaben, dazu eine **Prüfliste aus Code** (nicht aus Modellen): alle Antwortpflichten, wartenden Freigaben, laufenden Tasks, offenen Board-Items und unbeantworteten Nutzerfragen aus dem dauerhaften Zustand (`provider_reply_obligations`, `wait`, Async-Tools, Board).

Prüffragen:
1. Steht jeder Punkt der Prüfliste in "Offene Punkte"?
2. Fehlt eine Zusage, Korrektur oder Entscheidung des Nutzers, die im Original steht?
3. Enthält die Summary etwas, das im Original nicht steht (erfunden)?
4. Widerspricht etwas dem Gedächtnis?
5. Ist ein wichtiger Fakt weder in der Summary noch im Gedächtnis gelandet?
6. Halten alle Abschnitte ihr Budget?

Selbstkorrektur (automatisch, ohne Nutzer):
- Jeder Fund geht mit Beleg an die zuständige Rolle zurück, die liefert eine Korrektur. Höchstens zwei Korrekturrunden.
- Fehlt ein Punkt der Code-Prüfliste, fügt Code ihn direkt wörtlich aus der Quelle ein (kein Modell nötig).
- Fehlt ein Fakt im Gedächtnis, schreibt R3 ihn nach.
- Bleiben nach zwei Runden Funde: Rückfall auf das Hauptmodell mit strukturierter Anweisung (gleiche Abschnitte) über den bestehenden Pfad, dann einmal prüfen. Bleiben danach Funde, wird trotzdem committet, aber "Offene Punkte" stammt dann komplett aus der Code-Prüfliste plus wörtlichen Zitaten, und das Dashboard alarmiert.
- Jeder Lauf wird mit allen Ausgaben in einer Tabelle `rebuild_runs` gespeichert (Betriebsprojektion, 90 Tage), für Dashboard und Fehlersuche.

---

## 5. Einbau in Comma

### Elixir-Seite
- Neues Modul `SalixAgent.Rebuild` (Crew, Verifier, Zusammenbau). Es wird über die bestehende Seam in `SalixAgent.Compaction.summarize/1` (`SA/compaction.ex:449-474`) als neuer Zweig aufgerufen, der den vollen `plan` bekommt, inklusive `agent_id`, `session_id`, `tenant_id`, `opts[:runtime_session_config]` und Session. Die alte zweistellige `summarizer`-Signatur bleibt für Tests.
- Ergebnis an den Kernel als `{:summary, "<compacted-context>…</compacted-context>"}`.
- Die Crew-Modelle werden als normale Modell-Templates registriert (Chat-Completions an Together bzw. OpenRouter) und über `SalixAgent.LLM.complete` mit eigenem Metering-Entrypoint `rebuild` aufgerufen (Muster `metering_opts/4`, `SA/compaction.ex:584-603`). Konfiguration: `rebuild.models.crew`, `rebuild.models.verifier_1`, `rebuild.models.verifier_2` verweisen auf Template-Namen.
- Läuft als `DependencyJob(:compaction)` wie heute (`SA/internal_session_actor.ex:2071-2096`), Timeout für den ganzen Rebuild 5 Minuten.
- Strategie `openai_responses` (Provider-seitige Compaction) wird für den Router abgeschaltet, sonst greift die Seam nicht.

### Modell-Config: Cache-TTL
Neues Feld `cache_ttl_seconds` plus `cache_ttl_mode` am Modell-Template, durchgereicht an:
1. `CommaWeb.ModelTemplates` (`systems/apps/comma_web/lib/comma_web/model_templates.ex:6,79-119,22-49`),
2. `SalixAgent.Templates` (`SA/templates.ex:471-472,764-765,798-799`),
3. `llm_config_for_template/2` (`SA/templates.ex:837-866`),
4. `SalixAgent.AccountPool.resolve_config` (beide `Map.take`-Listen, `SA/account_pool.ex:1655-1657,1678-1681`),
5. `SalixAgent.Compaction` `@config_fields` und `kernel_config` (`SA/compaction.ex:71-81,540-544`), Kernel `CompactionHost.configOf` (`CompactionHost.lean:163-171`),
6. `SalixLlm.ProviderConfig.resolve` (`systems/apps/salix_llm/lib/salix_llm/provider_config.ex:63-65`), `Provider.Request.config` und `mark` (`VK/Provider/Request.lean:6-26`), damit Anthropic `cache_control` mit `ttl: "1h"` bekommt,
7. Dashboard-Formular (`systems/apps/salix_web/lib/salix_web/dashboard/live/template_live/form.ex`).

Standardwerte pro Anbieter (E35: die Logik nutzt pro Anbieter dessen Cache-Dauer und Mechanismus):
| Anbieter | Einstellung im Request (`cache_ttl_mode`) | `cache_ttl_seconds` | Keep-alive (Abschnitt 6) |
|---|---|---|---|
| Anthropic (API-Key und Abo) | `cache_control: {type: "ephemeral", ttl: "1h"}` an allen Breakpoints | 3600 | Gleicher Request wie die letzte Runde, `max_tokens: 0`, ohne Streaming, gleiche `thinking`- und `effort`-Werte |
| OpenAI bis GPT-5.5 | `prompt_cache_retention: "24h"` | 3600 | keiner |
| OpenAI ab GPT-5.6 | `prompt_cache_options: {ttl: "30m"}` (einziger erlaubter Wert) | 1800 | keiner |
| Alle anderen (DeepSeek, OpenRouter, Together) | nichts setzen | 300 | keiner |

Begründungen:
- 1 Stunde bei Anthropic: Mit 5 Minuten würde jede kurze Pause einen vollen Cache-Miss und über 100k einen Rebuild auslösen. Das Schreiben kostet mit 1 h das Doppelte statt das 1,25-fache, aber Qualität geht vor Kosten (E84), und weniger Rebuilds heißt weniger Verdichtung.
- 3600 bei OpenAI bis 5.5, obwohl der Cache bis 24 h halten kann: OpenAI garantiert die 24 h nicht, ein Miss kostet dort nur den normalen Eingabepreis (kein Schreibaufschlag), und so läuft der Rebuild bei beiden großen Anbietern nach derselben Pause. Sonst würde der Kontext tagsüber bis zur Notbremse wachsen.
- Kein Keep-alive bei OpenAI: Es gibt kein Gegenstück zu `max_tokens: 0`, jeder Call würde Ausgabe und Reasoning erzeugen.

### Prompt-Anpassungen
- Abschnitt "## History and memory" im Router-Prompt (`SA/tool_policy.ex:211-213`, "Compaction can remove relevant context") ersetzen: "Ältere Einzelheiten stehen nicht mehr wörtlich im Kontext. Nutze `memory.search`, `history.search` und `memory.source`, bevor du sagst, dass du etwas nicht weißt."
- Der Nutzer bekommt keinen Hinweis auf Rebuilds. Kein `clear`-Befehl im Client (der Befehl bleibt technisch für den Admin).

### Archiv-Retry (V14)
- Scheitert die Archivierung nach einem Commit, wird die Session in eine kleine Tabelle "Archiv ausstehend" eingetragen. Ein Oban-Job alle 10 Minuten verarbeitet nur diese Einträge (keine Vollscans) und ruft `InternalSessionStore.archive_compacted/2` erneut.

### Ausfall (V13)
- Scheitert der Rebuild, bleibt die alte Summary gültig, nichts geht verloren. Bestehender Backoff (`Query/Compaction.lean:329-353`: 30/120/300/600 s) gilt. Der nächste Versuch kommt mit dem nächsten Cache-Ablauf oder der Notbremse.

---

## 6. Cache warm halten, während der Router auf Helfer wartet

- Wartet der Router (`wait_for`, `SA/tools/async_ops.ex:95-141`) und arbeitet ein Helfer noch (`SalixAgent.WaitExtension.busy?/3`, `SA/wait_extension.ex:36-53`), sendet der Session-Actor kurz vor Ablauf der TTL (TTL minus 30 s, gerechnet ab letzter Cache-Berührung) einen Keep-alive-Call.
- Ort: prozesslokaler Timer in den Effekten `{:set_timer, :wait, _}` und `{:set_timer, :wait_for, wait}` (`SA/internal_session_actor.ex:1415-1428`), nach dem Muster `schedule_llm_retry` (`:2439-2451`). Abbruch bei `handle_cast(:wake)` und `request_process`.
- Der Call nutzt exakt denselben Präfix wie eine normale Runde. Neue Kernel-Query `keepalive_request` = `roundRequest` (`VK/Session/LoopHost.lean:548-569`) ohne Delta und ohne Tail. Ausführung als `DependencyJob(:llm)` über `SalixAgent.LLM.complete` mit Entrypoint `keepalive`, Form pro Anbieter laut Tabelle in Abschnitt 5 (nur Anthropic hat einen Keep-alive; bei allen anderen startet der Timer gar nicht). Kein Transkript-Commit. Der Actor merkt sich den Zeitpunkt als Cache-Berührung.
- Begrenzung: höchstens bis zur Warte-Decke von 1800 s (`systems/config/config.exs:641`). Mit 1 h TTL bei Anthropic ist das praktisch nie nötig, höchstens ein Call pro Warten.
- Im PR begründen: Der Keep-alive ist kein Loop-Request im Sinn von `RequestBound.requests_le_grants` (Host-Modell `VKP/Loop/Host.lean:40-63`), weil er keine Runde ist und nichts committet.

---

## 7. Gebundene Thinking-Blöcke (Anthropic ab Opus 5.5)

Gilt, sobald das Hauptmodell ein Anthropic-Modell ist. Paket P9 in `16` (Phase 1) baut alle Schritte. Das Feld `thinking_cutoff_id` aus Schritt 3 setzt P9 schon bei der heutigen Compaction, weil auch sie ein behaltenes Ende mit Thinking hat. P11 Rebuild nutzt dasselbe Feld. Header und `drop_block` setzt bereits Phase 0 (`16`), damit der Ist-Zustand nicht mit Fehler 400 scheitert.

### Fakt
Bei Claude Fable 5.1, Opus 5.5, Sonnet 5.5 und Haiku 5.5 ist jeder `thinking`-Block an den Präfix gebunden, der ihn erzeugt hat: `system`, die Tool-Menge und alle Nachrichten davor, byte-genau. Ändert sich davon etwas zwischen zwei Requests, ist jeder spätere Thinking-Block ungültig. Für Anthropic-Konten, die ab 31.08.2026 angelegt wurden, lehnt die API das mit Fehler 400 ab, auf allen Plattformen. Ältere Konten werden nicht geprüft, außer das Feld unten ist gesetzt. Opus 5.5 kann Thinking nicht abschalten (nur `thinking: {type: "adaptive"}`). Quelle: Skill `claude-api`, `shared/model-migration.md` "Breaking change 3", und `shared/preserved-thinking-migration.md`.

### Wo Comma den Präfix heute ändert (Stand `ba4f55d`)
| Stelle | Was sich zwischen zwei Requests ändert |
|---|---|
| `platformPrompt` (`VK/Session/Request.lean:178-180`) | Bei Herkunft außerhalb von `internal` (Telegram) hängt ein Plattform-Hinweis am `system`-String. Wechselt der Nutzer zwischen App und Telegram, ändert sich `system`. |
| Tail-Reminder (`Request.lean:282-291`: `obligationReminder`, `turnMarker`, alte Reminder) | Pro Request gebaut, nie gespeichert. Im nächsten Request fehlen sie an der alten Stelle. Betrifft jede Tool-Runde. |
| Miniskill-Overlays (`Request.lean:270-279`) | Pro Request aus der Auswahl gebaut, nie gespeichert. |
| Anhänge (`VK/Session/Attachments.lean:146-149`, `inline` ab `:164`) | Bilder gehen nur nach dem Watermark `last_ack_message_id` inline raus, danach nicht mehr. |
| `project_knowledge` (`VK/Session/Query/Compaction.lean:49-52`) | Ältere Blöcke werden bei der nächsten Aktivierung ausgeblendet. |
| Prompt-Snapshot (`ReplyQuery.promptSnapshot`, `VK/Session/Query/Reply.lean:398-405`), Katalogänderung aus `06` | `system` ändert sich mitten in der Session. |
| Tool-Menge (Tool-Disclosure, Plugins, MCP) | `tools` ändert sich mitten in der Session. |
| Rebuild (Abschnitte 2 und 3) | Neue Summary im `system`, R4-Kürzungen im behaltenen Ende. |

Nicht betroffen: das Aktivierungs-Delta. Es wird mit der Runde gespeichert (`activationEvents`, `VK/Session/LoopHost.lean:303`, genutzt in `loopRecord` ab `:387`) und später unverändert wieder gesendet. Gleiches gilt für `personal_memory` (`05`), das bis zum Rebuild sichtbar bleibt.

### Schritte
1. **Messen.** Der Anthropic-Adapter sendet für die vier Modelle immer den Header `anthropic-beta: thinking-binding-controls-2026-08-01` und `thinking.block_binding.prefix_mismatch_behavior: "drop_block"`. Jede Antwort enthält dann `input_transformations`. Der Adapter speichert jeden Eintrag (`type`, `reason`, `path`) mit Session und Runde im LLM-Log. Danach eine echte Session fahren: mehrere Tool-Runden, Wechsel App und Telegram, ein Bild, ein Miniskill, ein Rebuild. Jeder Eintrag `prefix_binding_mismatch` zeigt auf eine Stelle der Tabelle oder eine neue. Vorgehen: Skill `claude-api`, Workflow `preserved-thinking-migration`.
2. **Stellen auf Append-only umbauen:**
   - **Plattform-Hinweis:** raus aus `system`. Er wird eine Laufzeit-Nachricht im Aktivierungs-Delta, nur wenn sich sein Text gegenüber dem zuletzt gesendeten ändert (Provider-State-Schlüssel `platform_reminder` mit Hash des Texts, Muster `time_context` in `SA/context_providers.ex:34-81`). Damit wird er gespeichert wie jedes Delta.
   - **Tail-Reminder und Miniskill-Bodies:** werden mit der Runde gespeichert, in der sie gesendet wurden. Laufzeit-Nachricht mit Flag `turn_scoped: true`, an genau der Position, an der sie gesendet wurden (hinter Delta bzw. hinter den Tool-Ergebnissen, vor der Assistant-Nachricht der Runde). Der Anthropic-Encoder (`VK/Provider/Messages.lean`) sendet jede solche Nachricht als `{"role": "system", "clear_at": "next_user_message", "content": ...}` mit Beta-Header `mid-conversation-system-clear-at-2026-08-21`. Frühere Kopien bleiben byte-gleich in der Historie; ausgeblendete Kopien kosten keine Eingabe-Tokens. Lehnt der Weg die Beta ab (Fehler 400 mit unbekanntem Beta-Namen): Text-Block hinter den `tool_result`-Blöcken in derselben User-Nachricht, frühere Kopien bleiben stehen. Alle anderen Encoder lassen gespeicherte `turn_scoped`-Kopien früherer Runden weg und senden wie heute nur die aktuelle.
   - **Anhänge:** Ein einmal inline gesendeter Anhang wird bei jeder späteren Anfrage gleich inline gesendet, bis zum nächsten Rebuild. `candidate` prüft dafür statt des Watermarks, ob die Nachricht nach `thinking_cutoff_id` liegt (Schritt 3; gab es noch keinen Rebuild, gelten alle). Das gilt für alle Anbieter, weil es auch den Cache stabil hält.
   - **`project_knowledge`:** gleiche Regel wie `personal_memory`, sichtbar bis zum nächsten Rebuild.
   - **Prompt-Snapshot und Katalog:** Die Migration in `ReplyQuery.promptSnapshot` läuft nur noch im Rebuild-Commit. Dazwischen bleibt der gespeicherte Snapshot byte-gleich. Eine Regeländerung, die sofort gelten muss, kommt bis dahin als Migrationshinweis ins Aktivierungs-Delta (bestehender Mechanismus `SA/migration_notice.ex`, neue `@version`, gespeichert wie jedes Delta). Der Rebuild baut `system` ohnehin neu und entfernt das alte Thinking (Schritt 3).
   - **Tool-Menge:** Änderungen zwischen zwei Rebuilds (Plugin verbunden, MCP-Server neu) laufen über `drop_block`, das Thinking ab der Änderung fällt einmal weg. Zeigt die Messung aus Schritt 1 mehr als eine Tool-Änderung pro Tag, dann Umbau auf die volle Tool-Menge bei Session-Start und Rebuild mit `defer_loading: true` für noch verborgene Tools und Änderungen als `tool_addition`/`tool_removal`-Blöcke in einer `role: "system"`-Nachricht (Beta `mid-conversation-tool-changes-2026-07-01`).
3. **Rebuild:** Der Commit des Rebuilds speichert `thinking_cutoff_id` = höchste Nachrichten-ID zum Commit-Zeitpunkt, als Feld im bestehenden Compaction-Commit (`CompactionHost.lean:461-490`), kein neues Event. Der Anthropic-Encoder (`Messages.lean:34-38`) sendet für Assistant-Nachrichten mit ID ≤ `thinking_cutoff_id` kein Thinking. Text und Tool-Calls bleiben. Weil der Rebuild nie mitten in einer Tool-Runde läuft (Abschnitt 1), geht kein Thinking einer laufenden Runde verloren.
4. **Produktion:** `drop_block` bleibt dauerhaft gesetzt (lieber einmal Thinking verlieren als Fehler 400). In Evals gilt `"error"`, damit jede neue Präfix-Änderung die Suite rot macht. In CI prüft der Präfix-Test (unten) dasselbe ohne API-Call.
5. **Überwachen:** Dashboard (`15`) zählt `thinking_dropped` pro Tag nach `reason`. `model_binding_mismatch` nach einem Modellwechsel ist erwartet. Jeder `prefix_binding_mismatch` ohne Tool-Änderung oder Deploy davor ist ein Fehler und erzeugt einen Dashboard-Alarm. Ein Deploy, der das Rendering bereits gesendeter Nachrichten ändert (zum Beispiel der erweiterte Zeitblock aus `12`), verwirft das Thinking genau einmal; das Dashboard markiert solche Einträge mit dem Deploy-Zeitpunkt als erwartet.
6. Gilt für jeden Weg, der Anthropic-Requests baut (API-Key und Abo-Konto).

### Kernel und Beweise
- Gespeicherte `turn_scoped`-Nachrichten, `thinking_cutoff_id` und `reasoning_drop_ids` laufen über bestehende Event-Arten (Laufzeit-Nachricht der Runde, Compaction-Commit). Kein neues Event, kein neuer Reducer. `scripts/check.sh` muss grün sein. TLA unverändert (Begründung wie Abschnitt 1, Punkt 11).

### Tests
- Präfix-Test (Kernel, `protocol` `neutral` und Anthropic-Encoding): Für zwei aufeinanderfolgende Requests einer Session sind `system`, `tools` und `messages` bis zu den neu angehängten Nachrichten byte-gleich. Fälle: Tool-Runde, Wechsel App zu Telegram, Bild, Miniskill, zweite Aktivierung.
- Rebuild-Test: Nach dem Rebuild enthält kein Assistant-Turn im behaltenen Ende Thinking, Text und Tool-Calls sind unverändert.
- Live-Eval (`15`, Suite "Rebuild"): eine Session über 20 Runden mit Rebuild im Modus `"error"` ohne Fehler 400.

---

## 8. Abnahme

- Kernel-Tests (Abschnitt 1) und `scripts/check.sh` grün, `make tla` grün.
- Thinking-Bindung (Abschnitt 7): Präfix-Test und Rebuild-Test grün, Live-Eval ohne Fehler 400.
- Elixir-Tests: Crew-Teilausfall, kaputtes JSON, Verifier-Fund mit Korrektur, Rückfall auf Hauptmodell, Code-Prüfliste füllt fehlenden Punkt, Größenwächter, Kollision mit Nutzer-Input, Timer-Weckung ohne Nachricht, Keep-alive nur während Warten.
- Eval-Suite "Rebuild" (`15`) grün, Fehlerinjektion 100 Prozent gefunden.
- Abnahmetest 6 aus `16`: 30 Tage, Summary wächst nicht.
