# 07 Proaktivität

Ziel: Mails und andere Ereignisse wecken den Assistenten zuverlässig. Ein Entscheidungsmodell filtert vorher billig, ob etwas still bleibt, im Hintergrund erledigt wird oder gemeldet werden muss. Wichtiges hört der Nutzer immer, ohne Tageslimit (E51 bis E58). Die Überwachung stirbt nie still.

Pfad-Abkürzungen: `sa/` = `systems/apps/`.

---

## Ist-Zustand (Stand `ba4f55d`)

### Quellen und Weg einer Mail
1. Composio-Trigger `GMAIL_NEW_GMAIL_MESSAGE` (`sa/comma_web/lib/comma_web/member_source_triggers.ex:17`), Composio pollt, höchstens eine Lesung pro 60 s (`:22,60-75`). Zusätzlich 15-Minuten-Kette (`sa/comma_web/lib/comma_web/member_source_ingest.ex:23-25`).
2. Gmail-Filter `in:inbox is:important to:me -category:promotions -category:social newer_than:7d`, `maxResults 15` (`sa/comma_web/lib/comma_web/recommendation_google_member_source.ex:12,21`).
3. Kalender, Drive, Slack werden alle 15 Minuten gepollt.
4. Items landen im Pool (`sa/comma_core/lib/comma/member_source_items.ex`, höchstens 400 pro Quelle, 5 Tage).
5. `ProactiveCheck` (`sa/comma_web/lib/comma_web/proactive_check.ex`): prüft Schalter und Budgets (`:94-109`), dann bewertet das **große Router-Modell** bis zu 24 Items ohne Tools, 45 s Timeout (`:286-307`, `sa/comma_web/lib/comma_web/recommendation_renderer.ex:192-266`). Nur `critical`/`high` gehen weiter.
6. `ConversationServer.mail_interaction/5` → `ConversationActor.apply_mail_interaction/4` (`sa/salix_im/lib/salix_im/conversation_actor.ex:1954-2042`) mit Budget-Buchung (`:1961-1969`), versteckte Evidenz an den Router (`mail_router_input`, `:1901-1951`, Präfix "A background reminder needs your decision. Nothing has been sent…"), `app_event` mit `proactive_automatic` (`:1993`).
7. Der Router entscheidet `proactive.act notify/quiet/snooze` und sendet nach Home oder, wenn die Desktop-App inaktiv ist, nach Telegram/WeChat (`resources/salix-system-files/skills/proactive/SKILL.md:36-48`, `sa/comma_web/lib/comma_web/proactive_delivery.ex:31-56`).

### Budgets
- `sa/salix_im/lib/salix_im/mail_interaction.ex:6-16`: Übergaben (`sent_at`) höchstens 12 pro 24 h; Meldungen (`notified_at`) höchstens 5 pro 24 h mit 30 Minuten Abstand, kritisch überspringt nur den Abstand (`:78`). Geprüft beim Festhalten von `notify` (`:176-184`), nicht beim Senden.
- Doku-Drift: `docs/identity-security.md:280` nennt "one hour apart, three per 24 hours".

### V39: Überwachung stirbt still
- Die Kette startet nur beim Öffnen von Home (`sa/comma_web/lib/comma_web/router.ex:2655`) oder beim Einschalten (`sa/comma_web/lib/comma_web/proactive.ex:58`). Kein Cron, kein Boot-Hook.
- Uniqueness `states: :incomplete` (schließt `executing` ein) bei `MemberSourceIngest` (`member_source_ingest.ex:10-13`) und `ProactiveCheck` (`proactive_check.ex:12-15`).
- Wird ein Pod mitten im Lauf hart beendet (`shutdown_grace_period: 30_000`, `systems/config/config.exs:233`), bleibt der Job `executing`. `Comma.ObanPlugins.OperationLifeline` (`sa/comma_core/lib/comma/oban_plugins/operation_lifeline.ex:73-152`) rettet nur Jobs mit `operation_id` und drei Recommendation-Worker. Jeder spätere Start kollidiert still mit der Uniqueness. Nach 3 Fehlversuchen wird ein Job verworfen, die Kette ruht ebenfalls.
- Oban 2.23.0 (`systems/mix.lock:64`). Das Standard-Plugin `Oban.Plugins.Lifeline` ist nicht konfiguriert und soll es auch nicht werden: Es hat keinen Worker-Filter und würde Operation-Jobs beim letzten Versuch verwerfen (`operation_lifeline.ex:3-8`).

### V50: eine erledigte Mail macht alle still
- `proactive_check.ex:179-181`: Bei `:mail_source_handled` oder `:mail_source_changed` für das eine geroutete Item setzt `settle(items, "quiet")` alle bewerteten Items (bis 24) auf still. Still-Items werden erst bei Inhaltsänderung neu bewertet (`member_source_items.ex:290-297,367-369,429-433`).

### Doppelte Ereignisse (bleibt unabhängig von Budgets)
- Pool-Fingerprint und Baseline, `already_routed?` (`proactive_check.ex:242-251`), Oban-Uniqueness, `request_id = "check:" <> sha256(key <> observation_id)` (`:392-394`) mit `MailInteraction.plan/4`, Idempotenz-Keys im Actor (`conversation_actor.ex:1985-1986,2002-2003`), `dedup_key` in `watch.c` (`:186-190`).
- Lücke: Bei Nicht-Gmail-Quellen ist jede Textänderung eine neue Observation (`sa/comma_web/lib/comma_web/proactive_routine.ex:191-197`).

### Sonstiges
- Ausschluss persönlicher und sozialer Themen nur im Selection-Prompt (`recommendation_renderer.ex:61-62`).
- `app_active` zählt nur Electron-Sessions der letzten 10 Minuten (`sa/comma_core/lib/comma/accounts/sessions.ex:208-220`).
- Ohne `decide`-Key weckt jedes Watch-Ereignis den Router (`resources/salix-system-files/skills/proactive/scripts/watch.c:165-191`).

---

## 1. V39 beheben: Überwachung läuft immer

1. **Rettung hängender Jobs:** `Comma.ObanPlugins.OperationLifeline` bekommt eine dritte Klasse `rescue_proactive_jobs/2` nach dem Muster `rescue_recommendation_jobs` (`operation_lifeline.ex:113-152`), Aufruf in `handle_info` (`:57-58`). Worker: `CommaWeb.MemberSourceIngest`, `CommaWeb.ProactiveCheck`, `CommaWeb.ProactiveNotebook`, `CommaWeb.ProactiveTask`. Cutoff 300 s (größer als Job-Timeout 120 s plus Shutdown-Frist 30 s). Gerettete Jobs gehen auf `available`, `max_attempts` wird wie bei Operationen angehoben. Ein Neustart ist sicher: `record` arbeitet mit `FOR UPDATE` in einer Transaktion, `route` ist über `request_id` idempotent.
2. **Robuster Start:** neuer Cron-Eintrag `{"*/15 * * * *", Comma.Workers.ProactiveChainReconciler}` in `systems/config/config.exs:227-231`. Der Reconciler läuft auf dem Peer-Leader, blättert über `comma_recommendation_profiles` JOIN `comma_workspaces` (Owner, `relevance_mode` nicht `generic`, `sa/comma_core/lib/comma/recommendations.ex:232-233`) in Seiten zu 500 (Folgeseite als neuer Job) und ruft für jeden Owner eine neue öffentliche Funktion `MemberSourceIngest.ensure_chain(group, owner)` ohne Session (heute ist `insert/2` privat, `:122`, und `enqueue/3` braucht eine Session). Die Uniqueness `:incomplete` sorgt dafür, dass eine laufende Kette nicht doppelt startet. Der Worker liegt in `comma_web`, die Cron-Konfiguration in `:comma_core`; im Umbrella-Release funktioniert das direkt, sonst über einen Port wie `:recommendation_runtime_mod`.
3. **Verworfene Jobs:** Der Reconciler startet auch Ketten neu, deren letzter Job nach 3 Versuchen verworfen wurde.
4. **Start beim Boot:** Der Reconciler läuft zusätzlich einmal 60 s nach dem Start der App.
5. **Kardinalität (AGENTS.md):** ein Reconciler-Lauf pro 15 Minuten, höchstens ein Insert-Versuch pro Owner-Profil und Lauf. Für einen Nutzer trivial, auch bei vielen begrenzt durch Seiten.
6. **Überwachung immer an (F21):** Der Schalter in den Einstellungen steuert nur noch, ob der Router melden darf. Sammeln und Bewerten laufen immer. Beim Ausschalten bleibt der Pool erhalten (kein neuer Baseline-Satz, V42).
7. **Alarm:** Das Dashboard zeigt den letzten erfolgreichen Sammellauf pro Quelle. Älter als 30 Minuten: Systemmeldung an den Nutzer (`15`).
8. **Regressionstest:** Job im Zustand `executing` mit altem `attempted_at` anlegen, Lifeline laufen lassen, Kette läuft weiter. Zweiter Test: verworfener Job, Reconciler startet Kette.

---

## 2. Budgets entfernen

Vollständige Änderungsliste:

**Code**
1. `sa/salix_im/lib/salix_im/mail_interaction.ex`: `@window_ms` und `@budgets` (`:12-16`), `automatic_budget/3`, `notification_budget/4`, `budget/5`, `spends/4`, `record_spend/4` (`:66-108`), die Zweige in `reserve/5` (`:127-131`), die Plan-Fehler (`:175-184`) entfernen. Kommentar `:6-11` anpassen. `automatic_present?` bleibt (wird für `:proactive_disabled` gebraucht). Alte Werte `metadata.proactive.<owner>.sent_at/notified_at` bleiben als tote, begrenzte Daten liegen; keine Migration.
2. `sa/comma_web/lib/comma_web/proactive_check.ex`: Moduldoc `:8-10`, `cond`-Zweige `:99-109`, Deferral-Block `:120-129`, Zweig `:172-173` entfernen. `:deferred` bleibt für unklare und ungültige Bewertungen.
3. `sa/comma_web/lib/comma_web/home_mail.ex`: in `status/3` die Felder `automatic_budget` und `notification_budget` (`:25-28`) und `budget/1` (`:44-45`) entfernen.
4. `sa/comma_web/lib/comma_web/proactive_briefing.ex`: Gate `:32-40` und Moduldoc `:9-10`.
5. `sa/comma_web/lib/comma_web/proactive_task.ex:75`: Zweig `{:error, :proactive_budget_exhausted}`.
6. `sa/comma_web/lib/comma_web/proactive_notebook.ex`: Facts `notifications:`/`urgent:` (`:128-129`), Doku `:207-210`, `status_line/2` (`:244-253`), Texte `budget_open/urgent/closed` (zh-CN `:453-455`, en `:501-503`).
7. Kommentare und Beschreibungen: `sa/comma_web/lib/comma_web/proactive.ex:8-9`, `proactive_watch.ex:64`, `sa/salix_agent/lib/salix_agent/tools/proactive.ex:27`, `sa/salix_im/lib/salix_im/conversation_actor.ex:627-628,1900-1901`.

**Skill und Doku**
8. `resources/salix-system-files/skills/proactive/SKILL.md`: Überschrift `:26` ("The bar and the budget"), Absatz `:52-63`, `:83-84`, `:165`. Danach `devtools/salix-prompt-editor/data/catalog.generated.json` neu generieren.
9. `docs/identity-security.md:280` (heute schon falsch) streichen.
10. `docs/architecture/DOMAIN_CONCEPTS.md:1307,1315,1322-1327,1363` anpassen.

**Tests**
11. `sa/salix_im/test/mail_interaction_test.exs` (Tests ab `:47`, `:64`, `:119`), `sa/comma_web/test/local_recommendation_flow_test.exs` (`:1314-1381`), `sa/comma_web/test/proactive_mail_test.exs:2345-2355`, `sa/comma_web/test/proactive_notebook_test.exs:131-148`: Budget-Erwartungen entfernen, neue Erwartung "beliebig viele kritische Meldungen am Tag werden zugestellt".

**Bleibt bewusst**
- Das Loop-Budget `agent.notify` (6 Weckungen pro 10 Minuten für agentengeschriebene Loops, `sa/salix_agent/lib/salix_agent/loops.ex:13,46-60`) ist ein Schutz gegen fehlerhafte Loops, kein Meldebudget. Es bleibt.
- Alle Dedupe-Mechanismen bleiben. Das ist der Schleifenschutz: Dasselbe Ereignis wird nie zweimal gemeldet, verschiedene Ereignisse immer.
- Für die Dedupe-Lücke bei Nicht-Gmail-Quellen bekommt das Weck-Gate als Eingabe, wie oft und wann dasselbe Matter schon präsentiert wurde. Ist nur Kontext hinzugekommen, entscheidet es meist `still`.

---

## 3. Weck-Gate

### Ablauf in `ProactiveCheck`
Neue Stufe in `judge_and_route/7` (`proactive_check.ex:117-196`) vor der Bewertung durch das große Modell (`judge/5`, `:286-307`):
1. **Harte Regeln** pro Item (`14` Abschnitt 2): bekannte wichtige Person, Frist-Wörter, Termin in 48 Stunden, Antwort auf einen Thread, in dem der Assistent geschrieben hat. Treffer: direkt `melden`, ohne Modell.
2. **Gedächtnis-Kontext (V48):** Pro Item eine Gedächtnissuche nach Absender und Betreff (Top 5 Fakten, höchstens 2 KB), damit das Gate weiß, wer das ist und worum es geht.
3. **Entscheidungsmodell** `wake` (`14`): Choice `still`, `hintergrund`, `melden`, dazu Noul "Enthält das Item eine Frist oder einen Termin in der Zukunft?". Höchstens 3 Items pro Call (12-KiB-Grenze, Items bis 3,6 KB), Calls parallel innerhalb der Rate-Grenze (4 pro Sekunde, `decide/limits.ex`).
4. **Ergebnis:**
   - `still`: `settle(item, "quiet")` mit Grund. Bleibt im Briefing und auf dem Board als Vorschlag, falls das Briefing es wählt.
   - `hintergrund` oder `melden`: weiter zur Bewertung durch das große Modell wie heute, mit Gate-Ergebnis als Zusatzinfo. Danach Übergabe an den Router mit Dringlichkeit (`melden` → mindestens `high`).
   - Ausfall des Modells: `hintergrund` (der Router entscheidet).
5. Das große Modell bewertet nur noch, was das Gate durchlässt. Das ersetzt den heutigen Bewertungsaufruf für alle 24 Items.

### V46: Dringendes nicht abschneiden
Die harten Regeln laufen über alle Pending-Items, bevor die 24 für die Bewertung ausgewählt werden. Regeltreffer kommen zuerst in die Auswahl.

### V43: Fristen neu bewerten
Sagt das Gate `still`, aber Frist ja: Code liest das Datum (Kalender-Feld oder `dateparser` auf dem Text, de/en) und setzt am Pool-Item ein neues Feld `revisit_at` = Frist minus 24 Stunden. `ProactiveCheck` nimmt Items mit fälligem `revisit_at` wieder auf, auch ohne Inhaltsänderung.

### V50 beheben
`proactive_check.ex:179-181`: Bei `:mail_source_handled` oder `:mail_source_changed` nur das betroffene Item auf still setzen. Die anderen bleiben pending, und `ProactiveCheck` wird sofort neu eingereiht (nicht erst in 15 Minuten). Regressionstest: zwei dringende Items im selben Lauf, das erste wurde inzwischen bearbeitet, das zweite wird gemeldet.

### V49
Unklare Bewertungen nach drei Versuchen: statt liegenzulassen einmal mit dem Hauptmodell bewerten. Bleibt es unklar, geht das Item als `hintergrund` an den Router.

### V51: Quelle vor Meldung prüfen
Vor der Übergabe an den Router den aktuellen Stand der Quelle lesen (Gmail: Thread-Status, schon beantwortet? Kalender: Termin noch da?). Für Gmail existiert das schon im Übergabepfad. Für Kalender und Slack ergänzen.

### V41: Erstverbindung
Beim ersten Verbinden einer Quelle setzt Comma eine Baseline (keine Meldung für Altbestand). Neu: Die harten Regeln laufen einmal über die Baseline-Items. Treffer (offene Frist, unbeantwortete Mail einer wichtigen Person) gehen als `hintergrund` an den Router, der sie gesammelt in einer Nachricht nennt, falls wichtig.

---

## 4. Gmail-Filter und Quellen

- Filter (`recommendation_google_member_source.ex:12`) neu: `in:inbox newer_than:7d -category:promotions -category:social -category:forums`. `is:important` und `to:me` fallen weg, das Gate entscheidet (V59). `maxResults` von 15 auf 25.
- Composio-Trigger bleibt (Latenz etwa eine Minute ist für den Zweck ausreichend). Kalender, Drive, Slack bleiben bei 15 Minuten Polling.
- Persönliche und soziale Themen nicht mehr ausschließen (V55): Satz in `recommendation_renderer.ex:61-62` streichen. Es ist ein persönlicher Assistent.
- Watches (`watch.c`) laufen über den Entscheidungs-Router (`14`) mit kalibrierten Werten. Da `decide` jetzt konfiguriert ist, weckt nicht mehr jedes Ereignis den Router.

---

## 5. Was der Router mit Ereignissen macht

- Drei Stufen (V53): `still` erreicht den Router nicht. `hintergrund`: Der Router erledigt, was er ohne Rückfrage darf (Entwurf vorbereiten, Board-Todo anlegen, Information merken), und meldet sich nur, wenn der Nutzer etwas wissen oder entscheiden muss. `melden`: Der Router meldet sich in der passenden Form (`06`), oft ein Satz.
- Handeln statt Angebot (F18): Der Router startet erlaubte Arbeit direkt (Helfer, Entwurf), statt "Soll ich …?" zu fragen. Unumkehrbares läuft über das Aktions-Gate (`10`).
- Die versteckte Evidenz (`conversation_actor.ex:1901-1951`) bekommt das Gate-Ergebnis, die Gedächtnis-Treffer und die Anzahl früherer Präsentationen dazu.
- `SKILL.md` "Write the message" folgt den Formen aus `06`. Keine Pflichtfrage am Ende (V19).
- Lernen aus Feedback (F10): Ignoriert der Nutzer eine Meldung oder sagt "sowas nicht mehr melden", entsteht ein Fakt der Kategorie `vorliebe` ("Meldungen über X nicht nötig"), den das Gate über die Gedächtnis-Suche sieht. Dazu dient es als Label (`14`).

---

## 6. Briefing (V54, V56)

- Der Briefing-Input (`recommendation_renderer.ex:302-325`) bekommt den Kernblock (`04`), offene Board-Items und die Top-Fakten zu den Quellen.
- Die Übergabe an den Router (`proactive_briefing.ex`) enthält statt der Top 3 eine kompakte Liste aller gewählten Zeilen. Sie landen ohnehin als Vorschläge auf dem Board (`09`).
- Sprache: Deutsch (`12`).

---

## 7. Zustellung

Siehe `13_handy_und_voice.md`: iPhone-Push für Meldungen, iPhone zählt als aktiv, Telegram als Ausweichweg, ein bevorzugter Kanal statt Rundversand.

---

## 8. Abnahme

- Tests: V39 (hängender und verworfener Job), V50, keine Budgets (zehn kritische Meldungen an einem Tag werden alle zugestellt), Dedupe (gleiches Ereignis zweimal, eine Meldung), harte Regeln, Gate-Ausfall, `revisit_at`, Erstverbindung.
- Eval-Suite "Proaktivität" (`15`) grün.
- Abnahmetests 1, 3 und 5 aus `16`.
