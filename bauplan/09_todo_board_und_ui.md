# 09 Todo-Board und UI im Chat

Ziel:
1. Rechts ein Todo-Board statt der heutigen Task-Ansicht. Eine einzige Liste ohne Stufen-Tabs. Die Haupt-AI kennt es und pflegt es über ein unsichtbares Tool. Der Nutzer kann abhaken, und die AI weiß es im nächsten Turn (E68, E69).
2. Die Vorschläge der linken Leiste wandern als markierte Vorschläge aufs Board (E71).
3. Fragt der Assistent nach Datum oder Uhrzeit, kommt ein Picker im Chat (E72).
4. Wiederkehrende UI sieht gleich aus und wird nicht jedes Mal neu programmiert (E74). Ihr Zustand liegt auf dem Server, die AI sieht ihn.

Pfad-Abkürzungen: `sa/` = `systems/apps/`, `cl/` = `clients/`.

---

## Ist-Zustand (Stand `ba4f55d`)

### Leisten auf Home
- `cl/packages/app/src/components/RouteScreens.tsx:536-559,566-645`: links `HomeRail name="greet"` mit `RecommendationRail`, rechts `HomeRail name="tasks"` mit `HomeTasksRail`. Rail-Namen fest `"greet"|"tasks"` (`cl/packages/app/src/components/shellGeometry.ts:33`). Fold-Logik in `cl/packages/app/src/components/home/HomeRailFolds.tsx`.
- **HomeTasksRail** (`cl/packages/app/src/components/home/HomeTasksRail.tsx`): zeigt immer genau einen Status-Bucket (`in_progress`, `needs_review`, `backlog`, `done`, `cancelled`, `:17-23`) mit Umschalter, springt automatisch zu neuen Tasks. Daten aus der ProductInbox-Projektion (`tasks/useWorkspaceTasks.ts`), Reihenfolge über `GET/PUT /v1/comma/groups/:group_id/task-order[/:bucket]`. UI-Komponente `cl/packages/ui/src/components/home-tasks/HomeTasks.tsx` mit ungenutztem `onAddTask`. Buckets in `cl/packages/i18n/src/task-status.ts:11-57`.
- **RecommendationRail** ("Routines", `cl/packages/i18n/messages/en.json:105`): Eine Routine ist in Comma das Recommendation-Profil eines Mitglieds mit täglichem Briefing (Tabelle `comma_recommendation_profiles`, `sa/comma_core/lib/comma/data/recommendation_profile.ex:7-31`; `DOMAIN_CONCEPTS.md:654,680-790`). Inhalt: Karten mit Zeilen aus verbundenen Quellen. Ein Klick hängt den Prompt nur an den Composer-Entwurf an (`cl/packages/app/src/components/recommendations/appendRecommendationPrompt.ts:3-10`). Ausblenden lebt nur im React-State. Contract `cl/packages/recommendation-contract/src/index.ts`. API `GET …/recommendations`, `PATCH …/recommendations/settings`, `POST …/recommendations/refresh` (`sa/comma_web/lib/comma_web/router.ex:2836-2897`).
- iOS: `cl/apps/apple/iOS/HomeShellView.swift:426-476` (Task-Seitenleiste mit Buckets und `RoutinesStrip`).

### Domänenkonzepte
- Pins und Task-Order: Group-weit, nur Conversation-IDs (`sa/salix_im/lib/salix_im/conversation_group_actor.ex:21-24,307-333`). Nicht im Inventar.
- Agent Task: delegierte Arbeit, nur der Router legt an, Group-geteilt. Passt nicht für persönliche Todos.
- CalendarItem kennt `@type` `Event` und `Task` (`sa/salix_calendar/lib/salix_calendar/item.ex:4`). Lokale, von Menschen angelegte Items (`origin.kind=local`, unveränderlicher `owner_principal_ref`) unterstützen in v1 nur Event (`sa/salix_calendar/lib/salix_calendar/local_item.ex:3-10`). Lokale Items werden nicht zu externen Kalendern synchronisiert.
- "The device owns temporary UI state" (`docs/architecture/DOMAIN_CONCEPTS.md:344`).

### Rückfragen
- `question.request` geht nur in Telegram (`sa/salix_agent/lib/salix_agent/tools/async_ops.ex:69-71,161-162`), kennt nur Choices (`tools/schemas.ex:292-301`). Die interne Quelle hat die Fähigkeit nicht (`platform_capabilities.ex:24-25`).
- Capability-Requests haben keine Comma-Route und keinen Client-Konsumenten (`sa/salix_web/lib/salix_web/router.ex:2099-2145`, `capability_requests.ex:20`).
- Kein DatePicker in `@comma/ui` und iOS. `react-aria-components` ist Abhängigkeit (`cl/packages/ui/package.json:40-41`), `@internationalized/date` nicht.

### Dynamic UI
- `ui.create` nur für Worker (`sa/salix_agent/lib/salix_agent/tools/dynamic_ui.ex:59-64`), Payload unveränderlich in der Agent-VFS (`:752-777`), 11 Kartenarten (`cl/packages/chat-contract/src/dynamic-ui/cardContract.ts:395-407`), Zustand in localStorage, 32 KiB (`cl/packages/app/src/components/chat/dynamic-ui/DynamicUiWidget.tsx:103-111,284-380`). Neue Versionen starten leer. Nur Electron rendert. iOS zeigt nur die Zusammenfassung (`cl/packages/apple-core/Sources/CommaCore/Models.swift:341`; `cl/apps/apple/Shared/ChatComponents.swift:295-303` kennt `dynamic_ui` nicht).

---

## 1. Das Board

### Inhalt
Das Board ist eine Projektion über drei Quellen. Keine neue Entität für das Board selbst.

| Art | Quelle (Owner) | Beispiel |
|---|---|---|
| Todo | Lokales CalendarItem `@type Task` des Nutzers (siehe 1.1) | "Steuerberater Unterlagen schicken, bis Fr" |
| Läuft | Agent Task mit Status `active` (bestehend) | "Recherche Steuerberater" mit Zustand "läuft" oder "wartet auf Rechner" |
| Vorschlag | Zeile aus dem aktuellen Recommendation-Snapshot (bestehend) | "Auf Mail von X antworten" |

### 1.1 Persönliche Todos: CalendarItem `Task` erweitern
- Begründung (für `DOMAIN_CONCEPTS.md`): Ein persönliches Todo ist eine JSCalendar-Aufgabe (RFC 8984 `Task`: Titel, Fälligkeit, Fortschritt, Priorität). CalendarItem kennt den Typ schon, lokale Items haben schon einen unveränderlichen Owner. Eine neue Entität würde dieselbe Tatsache doppelt besitzen.
- `sa/salix_calendar/lib/salix_calendar/local_item.ex`: neben Event auch `@type "Task"` akzeptieren. Felder: `title` (≤ 512), `description` (≤ 4.000), `due` (optional, mit Zeitzone), `progress` (`needs-action`, `in-process`, `completed`, `cancelled`), `priority` (0 bis 9), `relatedTo` (Verweise: Agent-Task-Conversation-ID, Gedächtnis-Fakt-ID einer Zusage, Quell-URL), `x-comma-source` (`nutzer`, `assistent`), `x-comma-section` (`jetzt`, `spaeter`).
- Prüfung in `item.ex` `validate_object("Task", …)` erweitern. Verbotene Felder bleiben verboten.
- Speicher und Owner: der bestehende lokale Kalender-Store (`local_store.ex`, `local_items.ex`), Owner ist der Nutzer.
- Prüfe beim Bau, dass lokale Items nie in externe Kalender geschrieben werden. Falls doch (Sync-Pfad schreibt alle Items), dann statt dieser Erweiterung eine neue Entität "Personal Todo" mit Owner Nutzer anlegen und die Gründe im Inventar festhalten.
- Ein Todo mit `due` erscheint zusätzlich als Fälligkeit im Kalender des Nutzers in Comma. Der Router bekommt es als Erinnerung (bestehender Erinnerungsweg über Schedules), wenn es fällig wird.

### 1.2 Zusagen werden Todos (F13, V44, F15)
- Erkennt der Gedächtnis-Erfasser (`04`) oder Crew-Rolle R8 (`03`) eine Zusage (Kategorie `zusage`), legt Code ein Todo mit `relatedTo` auf den Fakt an.
- Zusagen des Assistenten ("ich melde mich morgen") werden Todos mit `x-comma-source=assistent` und `due`.
- Wiedervorlage: Ein Todo "auf Antwort von X warten" mit `due` weckt den Router zur Fälligkeit. Der Router prüft (z.B. Mail) und meldet sich nur, wenn nichts kam.

### 1.3 Unsichtbares Tool
Neue Router-Operationen `board.list`, `board.add`, `board.update`, `board.complete`, `board.remove`, `board.dismiss_suggestion`.
- Nur der Router. Worker melden an den Router.
- Werkzeugaufrufe sind im Chat nie sichtbar (Comma zeigt nur gesendete Nachrichten).
- Prompt-Regel: "Halte das Board aktuell. Lege ein Todo an, wenn der Nutzer etwas tun muss oder du etwas zugesagt hast. Hake ab, wenn es erledigt ist. Erwähne das Board nicht, außer der Nutzer fragt."
- Effektklasse `internal` (`10`), also nie ein Freigabe-Gate.

### 1.4 Board im Kontext des Routers
- **Änderungen seit dem letzten Turn** kommen als Teil des Aktivierungs-Deltas (`sa/salix_agent/lib/salix_agent/context_providers.ex:34-81`), angehängt, bricht den Cache nicht: "Nutzer hat abgehakt: …", "Nutzer hat hinzugefügt: …", "Task X läuft seit 12 min", "Vorschlag Y neu".
- **Vollständiges Board** steht im Abschnitt "Offene Punkte" jeder Summary (`03`) und ist per `board.list` abrufbar.
- Damit weiß der Router im nächsten Turn, was der Nutzer abgehakt hat (Abnahmetest 4).

### 1.5 Darstellung rechts (Desktop und Web)
- Ersetzt den Inhalt von `tasksContent` (`RouteScreens.tsx:553-559`). Rail-Name bleibt `tasks`, Geometrie bleibt.
- Neue Komponente `HomeBoard` in `cl/packages/ui` (Vorbild für Zeilen: `HomeTasks.tsx`), Daten über eine neue Projektion `board` (Server-Route siehe 1.6), live über die bestehenden Abo-Kanäle.
- **Eine Liste, keine Tabs, keine Bucket-Umschalter.** Abschnitte, alle gleichzeitig sichtbar:
  1. **Jetzt:** überfällige und heute fällige Todos, laufende Tasks, wartende Freigaben (als Verweis auf die Freigabekarte).
  2. **Später:** übrige offene Todos, sortiert nach Fälligkeit, dann Priorität.
  3. **Vorschläge:** Zeilen aus dem Recommendation-Snapshot, gedämpft dargestellt, mit Kennzeichnung "Vorschlag".
  4. **Erledigt:** in den letzten 24 Stunden erledigte Einträge, durchgestrichen. Danach verschwinden sie.
- Interaktion:
  - Todo: abhaken, Text bearbeiten, Fälligkeit setzen (Datum-Picker), löschen, neues Todo über ein Eingabefeld oben ("Neues Todo"), was das ungenutzte `onAddTask` ersetzt.
  - Laufende Task: Klick öffnet die Task-Ansicht (für Neugier, nicht Pflicht), "Abbrechen" (`08` Abschnitt 5).
  - Vorschlag: **"Machen"** schickt den Vorschlag direkt als Auftrag an den Router (nicht nur in den Entwurf, F18), **"Weg"** blendet ihn dauerhaft aus.
- Keine automatische Umschaltung, kein Springen.
- Reihenfolge automatisch (Fälligkeit, Priorität, Zeitpunkt). Kein Drag-and-Drop, damit kein neues Ordnungs-Objekt nötig ist.

### 1.6 Server
- Neue Comma-Routen unter `/v1/comma/groups/:group_id/board`: `GET` (Projektion), `POST /todos`, `PATCH /todos/:id`, `DELETE /todos/:id`, `POST /suggestions/:id/do`, `POST /suggestions/:id/dismiss`. Autorisierung: Owner der Group.
- `do` hängt eine Nutzernachricht in die Router-Conversation über `ConversationServer.append_group_conversation_message/3` (AGENTS.md: Mutationen nur über ConversationServer → ConversationActor). Text: der Prompt der Zeile.
- `dismiss` wird dauerhaft im Recommendation-Profil gespeichert (neues Feld `dismissed_item_keys`, begrenzt auf 500, ältere fallen raus), damit Ausblenden nicht mehr nur im React-State lebt.
- Änderungen erzeugen ein Ereignis auf dem bestehenden Conversation-Event-Strom (`GET …/conversations/events`, `router.ex:3239`) bzw. einem neuen Board-Strom, damit Desktop und iPhone live aktualisieren.

### 1.7 Linke Leiste
- Die linke Leiste (`RecommendationRail`) entfällt auf Home. Ihre Zeilen sind jetzt Vorschläge auf dem Board.
- Die Briefing-Einstellungen (Uhrzeit, Quellen) bleiben unter Einstellungen (`/#/settings?category=recommendations`).
- Das Briefing selbst läuft weiter und geht wie heute als Home-Matter an den Router (`DOMAIN_CONCEPTS.md:1366`). Der Router meldet daraus nur, was nach dem Weck-Gate (`07`) wichtig ist.
- Mit `greet` entfällt der linke Rail-Platz. `shellGeometry.ts:33` und `HomeRailFolds.tsx` auf eine rechte Leiste reduzieren. Das spart Platz für den Chat.

### 1.8 iPhone
- `HomeShellView.swift:426-476`: Bucket-Leiste durch dieselbe Board-Liste ersetzen (native SwiftUI, Abschnitte wie oben), `RoutinesStrip` entfällt.
- Abhaken, Hinzufügen, "Machen", "Weg" wie am Desktop.

---

## 2. Rückfragen mit Picker (Datum, Uhrzeit, Auswahl, Formular)

### Werkzeug
- `question.request` für die interne Quelle freischalten (`platform_capabilities.ex:24-25` um `"question"` ergänzen).
- Schema (`tools/schemas.ex:292-301`) erweitern: `input` = `choice` | `multi` | `date` | `time` | `datetime` | `text` | `form`. Bei `form` eine Liste von Feldern (`name`, `label`, `input`, `required`, `choices`). Dazu `timezone` (Standard: Nutzer-Zeitzone, `12`), `min`, `max`, `suggestions` (vorgeschlagene Termine als Schnellwahl), `locale`.
- Der Router darf die Frage stellen (nicht nur Worker).
- Ablauf asynchron: Der Router stellt die Frage und beendet den Turn. Die Antwort kommt als neue Nutzer-Eingabe mit strukturiertem Wert ("Antwort auf Frage q12: 2026-10-15T10:00+02:00") und startet einen neuen Turn. So wie die Telegram-Antwort heute als Provider-Input zurückkommt.

### Speicher und Weg
- Neuer Typ `question` in `@request_types` (`capability_requests.ex:20`), Speicher über `capability_request_store.ex`, unveränderliche Antwort, TTL 24 Stunden (danach "abgelaufen", Router erfährt es).
- Neue Comma-Routen: `GET /v1/comma/groups/:group_id/questions` (offene), SSE-Strom, `POST …/questions/:id/answer`. Autorisierung: Owner.
- Die Antwort wird als Nutzernachricht über `ConversationServer.append_group_conversation_message/3` in die Router-Conversation geschrieben, mit Metadaten `question_id`.

### Darstellung
- Chat-Contract: neuer Part `question` (`cl/packages/chat-contract/src/index.ts`), Swift-Wire neu generieren (`chat-contract/generated/CommaChatWire.generated.swift`).
- Desktop/Web: neue Komponenten in `cl/packages/ui`: `DatePicker`, `TimeField`, `DateTimePicker`, `ChoiceGroup`, `QuestionForm` auf Basis `react-aria-components`, Abhängigkeit `@internationalized/date` hinzufügen. Deutsche Formate (TT.MM.JJJJ, 24 h).
- iOS: SwiftUI `DatePicker`, `Picker`, `Form`.
- Nach Beantwortung zeigt die Karte die Antwort und ist gesperrt.
- Telegram (Ausweichweg): `choice` als Inline-Knöpfe (bestehend). `date`/`time` als Knöpfe für die `suggestions` plus "Anderes Datum" (Freitext).

---

## 3. Wiederverwendbare UI mit Zustand auf dem Server

### Router darf UI zeigen
- `ui.create` für `router` freigeben (`dynamic_ui.ex:59-64`). Der Router zeigt damit kleine Karten selbst, statt dafür einen Helfer zu starten.

### Vorlagen
- Neues Tool `ui.template.save`: macht aus einer bestehenden `ui_ref` eine Vorlage mit Name, Beschreibung und JSON-Schema für die Daten. Neues Tool `ui.show`: zeigt eine Vorlage mit neuen Daten. Validierung der Daten gegen das Schema wie bei `@card_schemas` (`dynamic_ui.ex:12-14`).
- Speicher: Vorlagen sind Ressourcen im Group-Scope des bestehenden Skill-Systems (Skills tragen Dateien, haben Scopes und werden in Runtime-Dateien projiziert, `DOMAIN_CONCEPTS.md:453,477-482`). Eine Vorlage ist ein Skill-Ordner `ui-<name>` mit `template.html`, `template.js`, `schema.json`, `SKILL.md` (Beschreibung, wann nutzen). Damit braucht es keine neue Entität, und der Router findet Vorlagen über den bestehenden Skill-Katalog.
- Prompt-Regel: "Bevor du UI neu baust, prüfe, ob eine Vorlage passt. Wenn du eine UI zum zweiten Mal ähnlich baust, speichere sie als Vorlage."
- Den "alten Helfer" erneut zu fragen ist nicht nötig. Jede Instanz nutzt die Vorlage.

### Zustand auf dem Server
- Ersetzt localStorage (`DynamicUiWidget.tsx:284-380`) als Quelle der Wahrheit. localStorage bleibt Cache.
- Owner-Entscheidung, die eine dokumentierte Regel umkehrt (`DOMAIN_CONCEPTS.md:344`): Der Nutzer will, dass die AI den Zustand sieht (E69, E74). Im PR festhalten, Inventar anpassen: "UI state for agent-visible widgets is owned by the Group and stored server-side; purely cosmetic state may stay on the device."
- Speicher: Zustandsdokument pro (Group, Widget-Schlüssel), höchstens 32 KiB, mit Revision (Compare-and-set). Widget-Schlüssel = Vorlagenname plus Instanz-ID, damit eine neue Version derselben Instanz ihren Zustand behält (heute starten neue Versionen leer).
- Owner und Mutation: `ConversationGroupActor` (hat schon Group-weite UI-Zustände mit CAS und Abo, `conversation_group_actor.ex:73-78,169-200,378-388`), neue Handler `put_ui_state`/`get_ui_state`.
- REST: `GET/PUT /v1/comma/groups/:group_id/ui-state/:key` neben den Pin-Routen (`router.ex:3230`).
- Agent-Tools: `ui.state.get`, `ui.state.set`.
- Rückweg von Widgets: Eine Änderung im Widget (Häkchen, Auswahl) schreibt den Zustand und erzeugt ein Delta-Ereignis für den Router ("Widget X: Feld Y = Z"), wie beim Board. Kein "Add to chat"-Umweg mehr nötig.

### iPhone und Web
- Web: `dynamicUiDocument()` von einem eigenen Sandbox-Origin mit CSP-Header ausliefern und `supportsDynamicUiWidgets` (`cl/packages/app/src/runtime-chat/nativePlatformActions.ts:3-5`) für Web freischalten. Vorher Sicherheitsprüfung des Origin-Aufbaus (Electron hat ein Request-Budget in `clients/apps/electron/src/main/security.ts:62+`, Web braucht ein Gegenstück).
- iPhone: `WKWebView` mit URL-Scheme-Handler, der dieselbe Runtime ausliefert, plus Swift-Message-Bridge für Zustand und Ereignisse. `ChatComponents.swift` um den Blocktyp `dynamic_ui` ergänzen. Board, Fragen, Freigaben und Reaktionen sind auf dem iPhone nativ (nicht Webview).

### Weitere Panels (später, nach dem Rest)
- Anheftbare Panels wie bei Hark: Ein Widget mit Zustand kann an die rechte Leiste angeheftet werden (unter dem Board). Panel-Pins mit stabiler `panel_id` im `ConversationGroupActor`, `ui.create` bekommt optional `panel_id`. Nur bauen, wenn Board, Picker und Vorlagen fertig sind.

---

## 4. Abnahme

- Tests: Todo anlegen, abhaken, bearbeiten über API und Tool; Delta erreicht den Router; Vorschlag "Machen" erzeugt Nutzernachricht; "Weg" bleibt nach Reload weg; Zusage erzeugt Todo; Frage mit Datum wird beantwortet und erzeugt Turn mit Wert; abgelaufene Frage; Vorlage speichern und mit neuen Daten zeigen; Zustand überlebt neue Version; Router liest Widget-Zustand.
- Playwright (`cl/packages/app/e2e`): Board ohne Tabs, Abhaken, Picker auswählen, Vorschlag ausführen.
- Abnahmetest 4 aus `16` (Todo auf zwei Geräten).
