# 08 Delegation und unsichtbare Helfer

Ziel: Der Router gibt Arbeit an Helfer ab, wenn das den Hauptkontext schützt oder parallel schneller ist. Der Nutzer sieht keine Helfer, keine Task-Karten, muss nichts abnehmen und nichts beaufsichtigen (E60, E61). Hängende Helfer fallen auf. Abbrechen funktioniert wirklich.

Pfad-Abkürzungen: `sa/` = `systems/apps/`, `cl/` = `clients/`, `cn/` = `systems/connector/salix-connect/`.

---

## Ist-Zustand (Stand `ba4f55d`)

### Delegationsregeln im Prompt
- `sa/salix_agent/lib/salix_agent/tool_policy.ex:175`: Delegieren ist Standard, "regardless of complexity or latency". `:179-185` erlaubt dem Router nur "one targeted read-only lookup". `:187`: Reviews brauchen eine Task. Reine Prompt-Regel (V21), der Router hat selbst `web.search` (`sa/salix_agent/lib/salix_agent/tools.ex:57`).
- `tool_policy.ex:177`: Pflichtkarte nach `im_api.internal.task.create` als Inline-`conversation_ref`.
- `sa/salix_im/lib/salix_im/provider/manuals.ex:697`: In `internal.send_message` muss jede Erwähnung einer bekannten Task als `conversation_ref` mit `presentation:"inline"` gesendet werden.
- `sa/salix_agent/lib/salix_agent/migration_notice.ex:187` (`content(16)`): aktiver Migrationshinweis mit derselben Pflicht. Aktuelle `@version` ist 52 (`:4`). Version 16 steht nicht in `@superseded_versions` (`:5-35`).
- `tool_policy.ex:123-133` (`@router_answer_delivery_prompt`): Der Router liefert Worker-Antworten aus. Das ist schon der Kern von "Ergebnis als normale Antwort".
- `tool_policy.ex:203-209` (`## Task outcomes`): Teilergebnisse oder offene menschliche Prüfung gehören nach `ready_for_review`. `completed` nur, wenn keine Arbeit, Entscheidung, Autorisierung oder Abnahme offen ist.

### Darstellung
- Chat-Contract-Part `inline-task` (`cl/packages/chat-contract/src/index.ts:129-161`), gerendert über `MessageInlineTask` (`cl/packages/app/src/components/chat/thread/inline/MessageInlineElements.tsx:276`) und `ConversationRefCard.tsx`.
- Task-Dock über dem Composer wird aus den `conversation_ref`s abgeleitet (`cl/packages/app/src/components/chat/tasks/useChatCreatedTasks.ts:49-99`). Ohne Refs bleibt er leer.
- iOS: `conversation_ref` wird zu `TaskReferenceCard` (`cl/apps/apple/Shared/ChatComponents.swift:319-327,352`).

### Abnahme
- Nur der Router setzt `ready_for_review` (`manuals.ex:508`). Annahme über `POST …/conversations/:id/accept` (`sa/comma_web/lib/comma_web/router.ex:3566-3579`) bis `do_accept_task_review_record` (`sa/salix_im/lib/salix_im/conversation_actor.ex:4306-4326`).
- Router-Selbstabschluss existiert für einfache Einmal-Tasks (`sa/salix_im/lib/salix_im/task_completion.ex:6-31`).
- Keine Review-Präferenz im Code.
- `ready_for_review` löst Push-Aufmerksamkeit aus (`sa/comma_core/lib/comma/notifications.ex:10-11`), blauen Punkt (`cl/packages/app/src/components/tasks/taskReviewAttention.ts:38`), Bucket `needs_review`.

### Parallelität
- Jede Task bekommt eine eigene Session (BEAM-Actor oder externer Connector-Prozess). Kein Limit pro Worker. Grenzen nur `DependencyAdmission` mit 64 global und 8 pro Tenant, je Node (`sa/salix_agent/lib/salix_agent/dependency_job.ex:233-239`).
- Keine Sandbox pro Task: VFS pro Agent, VM pro Group (`docs/architecture/DOMAIN_CONCEPTS.md:1054`), externe Runtimes nur ein Verzeichnis pro Session (`cn/external_runtime.go:970-1025`).
- Browser: Bindung pro Session, aber höchstens ein aktiver Browser pro Group (`sa/salix_store/lib/salix_store/browser_storage.ex:34-44`, Fehler `browser_shared_profile_in_use`). Parallele Browser-Tasks scheitern heute.
- Kein Lease für Desktop-Steuerung pro Task.

### Stillstand (V25)
- `sa/salix_im/lib/salix_im/task_worker_watch.ex`: reagiert nur auf `stopped`/`error` (`:245-253`). Jede Worker-Nachricht nach der Delegation schaltet ihn für diese Epoche stumm (`:374`). Ein Worker, der "bin dran" schreibt und dann hängt, wird nie gemeldet. Einzige Schranke: Router-Wartezeit bis 1800 s (`systems/config/config.exs:641`).
- Aktivitätssignale: intern `last_activity_at`, `activity_status_updated_at`, `activity_revision` (`sa/salix_agent/lib/salix_agent/internal_session/state.ex:49-53`). Extern nur `status_updated_at` bei Statuswechsel (`external_session_status.ex:495-516`), Connector-Events (`apply_runtime_event` `:113-180`) werden nicht als Fortschrittszeit projiziert.

### Abbrechen
- Nutzer-Abbruch lehnt der Server immer ab (`sa/comma_core/lib/comma/conversations.ex:514-520`).
- Router-Status `cancelled` stoppt keinen Worker.
- `fleet.ex:217-218` `abort_agent_runtime` ist Epochen-Fencing, kein Abbruch.
- Externer Stopp pro Session existiert: `ConnectedRuntimeDriver.stop` und `ComputeRuntimeDriver.stop` (`sa/salix_web/lib/salix_web/external_runtime.ex:157-168,202-224`), `cn/external_runtime_stop.go`.

---

## 1. Weiche Delegation

- Die harte Regel (`tool_policy.ex:175,179-187`) wird ersetzt durch: "Arbeite selbst, wenn die Anfrage mit wenigen Schritten erledigt ist und dein Kontext dabei nicht stark wächst. Gib ab, wenn die Arbeit viele Schritte, lange Recherche, große Tool-Ausgaben, Warten oder parallele Teile hat. Der Wert `delegation` in der Antwort-Richtung ist eine Richtung." Reviews dürfen beim Router bleiben, wenn alles Material sichtbar ist.
- Das Entscheidungsmodell `delegation` (`14`) läuft im selben Vor-Turn-Call wie die Antwortform (`06`). Eingabe zusätzlich: aktuelle Kontextgröße des Routers und Abstand zur Rebuild-Schwelle. Je größer der Kontext, desto eher `helfer`. Grund (E61): Mehr Arbeit im Hauptkontext heißt schnelleres Wachstum, mehr Rebuilds, schlechtere Qualität.
- Labels für die Kalibrierung: siehe `14` Abschnitt 3.

---

## 2. Unsichtbare Helfer

### Task-Karten weg (V24)
- `tool_policy.ex:177`: Pflichtkarte streichen. Neue Regel: "Erwähne Helfer und Tasks nicht. Sprich vom Ergebnis, nicht vom Weg. Wenn der Nutzer fragt, was läuft, antworte aus dem Board."
- `manuals.ex:697`: Die Pflicht, Tasks als Inline-`conversation_ref` zu senden, für Comma Home streichen. Ein `conversation_ref` ist nur noch erlaubt, wenn der Nutzer ausdrücklich nach einer Task fragt. Slack und Telegram bleiben unverändert (dort eigene Karten).
- `migration_notice.ex`: neue Notice-Version 53, die Version 16 in `@superseded_versions` aufnimmt und die neue Regel nennt.
- Die Reaktion ⏳ auf die Nutzernachricht ersetzt die Karte als sichtbares Zeichen (`06`).
- Clients: keine Änderung nötig, ohne Refs entsteht keine Karte und kein Dock-Eintrag. Laufende Helfer sieht der Nutzer auf dem Board (`09`).

### Ergebnis als normale Antwort
- Bleibt wie in `@router_answer_delivery_prompt` (`tool_policy.ex:123-133`), ergänzt um die Antwortform aus `06`: Der Router formuliert das Ergebnis in seiner Stimme, kurz, ohne "der Helfer hat herausgefunden".
- Vor "erledigt" prüft der Router, dass das Ergebnis da und zugestellt ist (V28): Datei existiert, Mail-Entwurf liegt vor, Termin steht im Kalender. Dafür reicht ein Lese-Tool-Call.

### Abnahme standardmäßig aus
- Owner-Entscheidung (beim Start bestätigt, `README.md` Frage 19), im PR dokumentieren (AGENTS.md: Schwächung einer bestehenden Garantie braucht die Entscheidung des Owners).
- `tool_policy.ex:203-209` neu: "Setze `completed`, sobald das Ergebnis zugestellt ist. `ready_for_review` nur, wenn der Nutzer eine Entscheidung treffen muss, die du nicht treffen darfst. Freigaben für unumkehrbare Aktionen laufen über das Aktions-Gate (`10`), nicht über die Abnahme."
- `task_completion.ex:6-31`: Router-Selbstabschluss für alle eigenen Tasks erlauben, nicht nur einfache Einmal-Tasks. Schedule-, Triage- und Workflow-Tasks behalten ihre eigenen Regeln.
- Die bestehende Annahme-Route bleibt für die seltenen `ready_for_review`-Fälle.

### Durcharbeiten statt Zwischenfragen (E54)
- Regel im Router- und Worker-Prompt: "Arbeite so weit wie möglich, bis eine Freigabe nötig ist oder die Aufgabe fertig ist. Frage nur bei echten Blockern (fehlende Information, die du nicht finden oder sinnvoll annehmen kannst). Triff vernünftige Annahmen und nenne sie kurz im Ergebnis."
- Worker fragen nie den Nutzer, sondern den Router. Der Router beantwortet aus Gedächtnis und Kontext oder sammelt mehrere Fragen zu einer einzigen Rückfrage (mit Picker, `09`).

### Helfer melden an den Router, nicht an den Nutzer
- Bleibt wie heute: Worker-Berichte kommen als Task-Nachrichten zum Router (`tool_policy.ex:58`).
- Worker bekommen Gedächtnis zum Start und Lese-Tools (`05` Abschnitt 3).

---

## 3. Parallele Helper und geteilte Geräte

- Mehrere Helfer laufen parallel, jeder in eigener Session. Daran ändert sich nichts.
- **Browser-Warteschlange:** Statt mit `browser_shared_profile_in_use` zu scheitern (`browser_storage.ex:34-44`), stellt sich eine zweite Task in eine Warteschlange pro Group (FIFO, Wartezeit höchstens 20 Minuten). Der Worker bekommt die Rückmeldung "Browser belegt, du bist an Position n" und wartet über `wait_for`. Der Inhaber gibt frei, wenn seine Session endet oder der Browser 60 s untätig ist (bestehendes `keep_alive`).
- **Desktop-Lease:** Für den Windows-Rechner des Assistenten (`11`) gibt es dasselbe Lease-Modell: genau eine Task besitzt den Desktop. Vorbild ist das Android-Lease (`sa/salix_agent/lib/salix_agent/tools/schemas.ex:1027-1034`, `sa/salix_web/lib/salix/control/android_control.ex:33-43`, `max_concurrent_leases == 1`). Lease mit `lease_seconds` 60 bis 3600, Verlängerung bei Aktivität, Warteschlange wie beim Browser.
- Browser und Desktop desselben Windows-Rechners sind ein gemeinsames Lease (der Browser läuft dort auf dem Desktop, `11`).
- Das Board zeigt wartende Tasks mit "wartet auf Rechner".
- Keine weiteren Rechner. Wenn die Warteschlange im Alltag stört (Dashboard: mittlere Wartezeit über 10 Minuten an mehreren Tagen), ist die Lösung ein zweites Browserprofil auf demselben Windows-Rechner, nicht ein zweiter VPS.

---

## 4. Stillstandserkennung (V25)

- `task_worker_watch.ex` `monitored/1` (`:245-253`) bekommt einen dritten Zweig: Zustand `active`, nicht in einem `wait` mit Restzeit (`sa/salix_agent/lib/salix_agent/session_activity.ex:64-72,129-151`), und letzter Fortschritt älter als `task_worker_stale_seconds` (Standard 600).
- Fortschritt ist: neues Kernel-Event (intern `activity_revision`), LLM-Call-Ende, Tool-Call-Ende, Connector-Event (extern).
- Externe Sessions: `ExternalSessionStatus.apply_runtime_event` (`external_session_status.ex:113-180`) schreibt bei jedem Connector-Event ein Feld `progress_at` in die Projektion. `session_activity.ex` nutzt es für `updated_at`.
- Die Stummschaltung durch Worker-Nachrichten (`:374`) gilt nur noch für den bisherigen Fall (Worker hat gestoppt). Für den Stale-Zweig zählt nur der Fortschritt.
- Fence für Idempotenz: Epoche plus Stale-Fenster-Nummer (gerundete Zeit des letzten Fortschritts), weil ein Hänger keine neue `version` erzeugt (`:31-40`).
- Ablauf: Nach 10 Minuten ohne Fortschritt ein `task_worker_nudge` an den Worker (bestehender Mechanismus). Nach weiteren 5 Minuten ohne Fortschritt ein `task_worker_stopped`-artiges Ereignis an den Router (versteckt, wie heute). Der Router entscheidet: neu starten, abbrechen (Abschnitt 5), oder den Nutzer kurz informieren, wenn es wichtig ist.
- Rate-Limit und Quota (V26): `rate_limited` und `quota_exhausted` sind schon blockierende Issues (`:66-73`). Neu: Wenn der Anbieter eine Reset-Zeit nennt, registriert der Watch einen dauerhaften Timer (`SalixStore.Timers`, neues Kind `worker_resume`) und weckt den Worker danach mit "weiter". Ohne Reset-Zeit: Router entscheidet.

---

## 5. Abbrechen

- Router-Status `cancelled` (`manuals.ex:508`) stoppt die Worker-Session wirklich:
  - Externe Session: `ConnectedRuntimeDriver.stop` bzw. `ComputeRuntimeDriver.stop` (`external_runtime.ex:157-168,202-224`).
  - Interne Session: neue Funktion `SalixAgent.Fleet.stop_session(agent_id, session_id, reason)`, die den Session-Actor stoppt, laufende `DependencyJob`s abbricht und die Session als beendet markiert. Nicht `abort_agent_runtime` verwenden (das ist Epochen-Fencing).
  - Browser- und Desktop-Lease der Session werden freigegeben.
- Nutzer-Abbruch: `Comma.Conversations.cancel/4` (`conversations.ex:514-520`) für `agent_task` zulassen und auf denselben Pfad führen. Auslöser im Client: auf dem Board-Item "Abbrechen" (`09`), und im Chat per Sprache ("stopp das mit X"), worauf der Router `cancelled` setzt.
- Wirkung, die schon passiert ist, wird nicht rückgängig gemacht (bestehende Aussage `tool_policy.ex:209`).

---

## 6. Abnahme

- Tests: keine Inline-Task-Refs in Home-Antworten; Router setzt `completed` nach Zustellung; Browser-Warteschlange statt Fehler; Desktop-Lease exklusiv; Stale-Erkennung intern und extern nach Fortschritt-Nachricht; kein doppeltes Nudge im selben Fenster; `cancelled` stoppt interne und externe Session; Nutzer-Abbruch über die Route; Rate-Limit-Timer weckt Worker.
- Eval-Suite "Delegation" (`15`) grün.
- Abnahmetests 8 und 9 aus `16`.
