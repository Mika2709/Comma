# 12 Sprache und Zeit

## Teil A: Alles auf Deutsch

Ziel (E82): Alles, was der Nutzer sieht, ist Deutsch: Oberfläche, Antworten, Briefings, Meldungen, Push, Telegram, E-Mails. Code-Namen bleiben Englisch. Englische Wörter, die der Nutzer selbst benutzt oder die im Deutschen üblich sind (Deployment, Pull Request, Meeting, Call, Board, Todo, Chat), bleiben Englisch.

Interne System-Prompts bleiben Englisch (bessere Befolgung, einfacher Upstream-Merge). Sie enthalten die Sprachregel.

Pfad-Abkürzungen: `sa/` = `systems/apps/`, `cl/` = `clients/`.

### Ist-Zustand
- Nur `en` und `zh-CN`: `sa/comma_core/lib/comma/accounts/user.ex:70-71` (`validate_inclusion` `:59`, `check_constraint :comma_users_locale_check` `:66`), DB-Constraint in `sa/comma_core/priv/repo/migrations/20261001190000_add_user_locale.exs:13`, Standard `"en"` (`accounts.ex:169-174`).
- Briefings fallen bei allem außer `zh-CN` auf Englisch zurück (`sa/comma_web/lib/comma_web/recommendation_renderer.ex:416-430`, V20).
- Client-Kataloge `cl/packages/i18n/messages/en.json` und `zh-CN.json` mit je 2.519 Keys, Laufzeit Paraglide.
- iOS: `cl/apps/apple/Resources/Localizable.xcstrings` (177 Einträge, en und zh-Hans).
- Login-E-Mails nur Englisch (`sa/comma_core/lib/comma/email_delivery.ex:7-60`).
- `SalixVoice.Profile` kennt Deutsch schon (`sa/salix_voice/lib/salix_voice/profile.ex:20-25`).

### Änderungen
**Backend**
1. `user.ex:71` um `de` erweitern, neue Migration ersetzt `comma_users_locale_check`.
2. Standard-Locale dieser Installation `de` (Config `comma.default_locale`, gelesen in `accounts.ex:169-174`). Das Konto des Nutzers auf `de` setzen.
3. `recommendation_renderer.ex:426-430` (`language/1` → "German", `output_language/1` → "de"), Prompt-Hinweis `:94` ("Aim for 8-15 words") sprachneutral formulieren.
4. `sa/comma_core/lib/comma/recommendation_draft.ex:150,390,396,424`, `recommendation_member_selection.ex:142-180` (Titel "Your work briefing" → "Dein Briefing").
5. `sa/comma_web/lib/comma_web/home_mail.ex:562-575` (`due_text`), `proactive_notebook.ex:447,493` (`text("de")`), `sa/comma_core/lib/comma/recommendations.ex:756-759`.
6. APNs: `sa/comma_core/lib/comma/notifications.ex:405-416` (`status_label`, `normalize_locale` um `de-DE`).
7. Telegram: `telegram_commands.ex:15,24,94`, `telegram_command_setup.ex:11,36`, `telegram_task_cards.ex:353-431`, `telegram_integration.ex:219,270`, `sa/salix_im/lib/salix_im/telegram_interactions.ex:44,735`.
8. Tool-Enum `sa/salix_agent/lib/salix_agent/tools/schemas.ex:27-29` (`["en","zh-CN","de"]`).
9. IFC-Fußzeile `sa/salix_agent/lib/salix_agent/ifc.ex:38-43,239-243` um `:de`.
10. `chat_device_view.ex:5`, `wechat_device_commands.ex:16`.
11. Login-E-Mails: Locale-Parameter einführen, deutsche Texte.

**Clients**
12. `cl/packages/i18n/messages/de.json` mit allen 2.519 Keys. Übersetzung durch die umsetzende AI mit dem Glossar unten, Du-Form, kurz.
13. Locale an allen vier Stellen registrieren: `project.inlang/settings.json:4`, `src/locale.ts:1` (`supportedLocales`), `resolveLocale` (`:8-29`, de-Mapping), `scripts/check-i18n.mjs:8`.
14. Native-Bridge-Enums `cl/packages/native-bridge/src/capability-leaves.ts:938,8252`, danach regenerieren (`pnpm --dir clients check:foundation`).
15. Einstellungen: Sprachauswahl `AppSettingsRoute.tsx:1161-1167`, neuer Key `settings_language_german`.
16. Hart codierte zh/en-Zweige: `AutomaticUpdates.tsx:180-209`, `SitePermissionMenuApp.tsx:74`, `billing/useUsageBillingCategory.tsx:49`, `application-menu/useApplicationMenu.ts:16`, Electron `src/main/application-menu.ts:37-70` und `application-menu-provider.ts:5`, `modules/browser-sidebar/site-permissions-platform.ts:17`.
17. iOS: `Localizable.xcstrings` um `de`, `Comma.xcodeproj/project.pbxproj:439-442` `knownRegions` um `de`, `iOS/PhoneAppDelegate.swift:28-30` `notificationLocale`, Test `Tests/TaskNotificationScreenshotTests.swift:17,60`.

**Prompts**
18. Sprachregel im Router-Prompt (`@router_message_language_prompt`, `sa/salix_agent/lib/salix_agent/tool_policy.ex:141-145`): "Write to the user in German. Keep English terms the user uses or that are common in German (e.g. deployment, pull request, meeting, call, board, todo). Do not force-translate them. If the user writes a whole message in English, answer in English." Gleiche Regel für Worker-Ergebnisse, die an den Nutzer gehen.
19. Skills unter `resources/salix-system-files/skills` sind sprachneutral ("owner's language", `proactive/SKILL.md:93-94`), bleiben.

### Glossar für die Übersetzung
| Englisch im Code/UI | Deutsch in der UI |
|---|---|
| Task | Aufgabe (im Board: läuft) |
| Worker | Helfer (in der UI kaum sichtbar) |
| Router / Assistant | Assistent |
| Home | Chat |
| Routine / Briefing | Briefing |
| Approval | Freigabe |
| Review | Abnahme |
| Board / Todo | Board / Todo |
| Workspace / Group | Arbeitsbereich |
| Device | Gerät |
| Settings | Einstellungen |
| Memory | Gedächtnis |
| Source | Quelle |

### Abnahme
- `node cl/packages/i18n/scripts/check-i18n.mjs` grün (gleiche Keys und Parameter in allen Locales).
- Playwright-Rauchtest mit Locale `de`: keine englischen Strings auf Home, Board, Einstellungen.
- Abnahmetest 12 aus `16`.

---

## Teil B: Zeit

Ziel (E23, E24, E83): Der Assistent weiß in jedem Turn, wann es ist und wie viel Zeit seit der letzten Nachricht vergangen ist, in der Zeitzone des Nutzers, unsichtbar für den Nutzer. Zeit ist ein Kernbegriff im Gedächtnis (`04`).

### Ist-Zustand
- Pro Nutzernachricht gibt es schon einen Zeitblock: `input_time` (`source_sent_at`, `received_at` in UTC, `timezone`, `timezone_source`, `anchor_kind`), erzeugt in `SalixAgent.InputTime.capture/1` (`sa/salix_agent/lib/salix_agent/input_time.ex:6-21`), gesetzt in `SalixAgent.deliver/3` (`sa/salix_agent/lib/salix_agent.ex:85-90`) und `ConversationConsumer.stamp_input_time` (`sa/salix_agent/lib/salix_agent/conversation_consumer.ex:77,337-340`), gerendert als "Message time context (source metadata, not user text): {json}" vor dem Nutzertext (`systems/native/verified_kernel/runtime/VerifiedKernel/Provider/Content.lean:28-33`). Er ist Teil der neuen Nachricht und bricht den Cache nicht.
- Für Chat-Nachrichten ist die Zeitzone unbekannt (`timezone: nil`, `timezone_source: "unknown"`). Nur Schedules setzen `source_timezone` (`sa/salix_cluster/lib/salix_cluster/schedules.ex:461-462`). Comma-Chat setzt nur `source_sent_at_ms` (`sa/salix_im/lib/salix_im/conversation_delivery.ex:84`).
- `TimeContext` (`sa/salix_agent/lib/salix_agent/time_context.ex`) frischt bei jeder neuen Nutzernachricht auf, sonst alle 5 Minuten, mit `clock_timezone: UTC` und der Anweisung, die Zeitzone nicht zu raten.
- Einziges Zeitzonenfeld: `Comma.Data.RecommendationProfile.timezone` (Standard `Etc/UTC`, aus der Geräte-Zeitzone gefüllt, `sa/comma_core/lib/comma/data/recommendation_profile.ex:13`, `recommendations.ex:726-796`).
- Tagebuch-Datum nach Server-Zeit UTC (`sa/salix_agent/lib/salix_agent/tools/memory.ex:550-559`, V04).

### Änderungen
1. **Zeitzone des Nutzers:** neues Feld `timezone` (IANA) am User (`Comma.Data.User`), Standard aus der Geräte-Zeitzone beim ersten Login, in den Einstellungen änderbar. Für diese Installation `Europe/Berlin`. `RecommendationProfile.timezone` liest künftig daraus (bleibt als Feld für Kompatibilität, wird beim Speichern synchron gehalten). Kein neues Domänenkonzept, nur ein Wert am User.
2. **Jede Eingabe bekommt die Zeitzone:** `conversation_delivery.ex:73-89` füllt `source_timezone` aus dem User. Dasselbe für Telegram-, Mail-, Erinnerungs- und Helfer-Ereignisse an den Router (Owner-Zeitzone).
3. **Zeitblock erweitern:** `InputTime.capture/1` ergänzt `local_time` (ISO mit Offset), `weekday` (deutsch), `since_previous_seconds` (Abstand zur vorigen Nachricht in dieser Conversation, berechnet beim Einliefern). `Content.inputTime` rendert kompakt, z.B. "Zeit: Do 08.10.2026 21:40 (Europe/Berlin), 9 h 12 min nach der letzten Nachricht". Das Modell sieht es in jedem Turn, der Nutzer nie.
4. **TimeContext:** `clock_timezone` = Nutzer-Zeitzone, die Anweisung "never infer user timezone" entfällt, wenn sie bekannt ist.
5. **Gedächtnis:** Tagebuch und Tagesseiten nach Nutzer-Zeitzone (`memory.ex:550-559`, V04). Hindsight bekommt Zeitstempel mit Offset, `dateparser` rechnet in der Nutzer-Zeitzone (`04`).
6. **Datumsrechnen per Werkzeug:** neues Tool `zeit.rechnen` (Router und Worker, Effektklasse `read`): Wochentag eines Datums, Datum plus/minus Tage/Wochen/Monate, Abstand zwischen zwei Daten, "nächster Donnerstag" ab heute, Umrechnung zwischen Zeitzonen. Deterministisch in Elixir. Prompt-Regel: "Rechne Daten nie im Kopf, wenn es auf den Tag ankommt. Nutze `zeit.rechnen`."
7. **Prompt-Abschnitt "Zeit"** im Router-Prompt: "Du kennst die aktuelle Zeit aus dem Zeitblock jeder Nachricht. Nenne Termine mit Wochentag und Datum. Sag, wie lange etwas her ist, wenn es hilft ('vor 3 Tagen'). Prüfe Fälligkeiten gegen das aktuelle Datum. Erinnerungen aus dem Gedächtnis tragen ihr Datum: Behandle alte Status-Infos als möglicherweise veraltet."

### Abnahme
- Tests: Zeitblock enthält Nutzer-Zeitzone und Abstand; Tagebuch um 00:30 Ortszeit landet im richtigen Tag; `zeit.rechnen` für Monatsgrenzen, Schaltjahr, Sommerzeitwechsel.
- Eval-Suite "Zeit" (`15`).
- Abnahmetest 13 aus `16`.
