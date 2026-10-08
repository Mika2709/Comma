# 06 Antwortform und Reaktionen

Ziel: Der Assistent antwortet wie ein guter Mensch im Chat. Oft nur eine Reaktion oder ein Satz, manchmal ausführlich, manchmal gar nichts. Keine Romane, kein Filler, aber auch keine weggelassene Kerninfo.

Grundsatz aus dem Gespräch: Ein hartes Limit macht es schlimmer (Filler bleibt, Wichtiges fällt weg, Token-Limits schneiden mitten im Satz ab). Wirksam sind drei Dinge zusammen:
1. Eine Entscheidung vor dem Turn, die eine grobe Richtung vorgibt.
2. Ein Sende-Tool, dessen Beschreibung bei jedem Aufruf einen zweiten Gedanken erzwingt.
3. Reaktionen als echte Antwortform.

Pfad-Abkürzungen: `sa/` = `systems/apps/`, `cl/` = `clients/`, `VK/` = `systems/native/verified_kernel/runtime/VerifiedKernel/`.

---

## Ist-Zustand (Stand `ba4f55d`)

- Text erreicht den Nutzer nur über `im_api.internal.send_message` oder `end_turn.reply` (`docs/tools-integrations.md:148-159`). Normaler Modelltext wird nie angezeigt. `reply` ist optional (`systems/apps/salix_agent/lib/salix_agent/turn_outcome.ex:10-21`). Ein Turn darf ohne Nachricht enden (`systems/native/verified_kernel/runtime/VerifiedKernel/Session/TurnReminders.lean:14,18`).
- Die `opening`-Regel (`TurnReminders.lean:16`) zwingt den Router, auf jede neue Comma-Nutzernachricht einen kurzen Satz zu senden. Eine reine Reaktion ist so unmöglich.
- Reaktionen gibt es nicht in Comma Home (`systems/apps/salix_im/lib/salix_im/provider/internal.ex:392` sendet nur), nicht in Telegram (`telegram.ex:238-242`), WeChat, iMessage. Es gibt sie in Slack (`operation_registry.ex:508`), Feishu (`:887`), Signal (`signal.ex:25,60`).
- Vorlage: Slack-Triage wählt schon `reply|reaction|silence` mit Emoji-Allowlist und 4000-Zeichen-Grenze (`triage/participation_decision.ex:18`, `triage/product_decision.ex:29,37`).
- Länge wird nur per Prompt gesteuert: `opening`-Regel, `resources/salix-system-files/skills/proactive/SKILL.md` ("Write the message": 1 bis 2 Sätze plus Frage), Aufmerksamkeitsnachricht bis 600 Zeichen (`comma_web/recommendation_renderer.ex:168-186`), Voice-`reply_length` per Jev (`docs/messaging-voice.md:191`).
- Einziger Engpass aller Sendungen: `ImRouter.call_dynamic_operation` (`systems/apps/salix_agent/lib/salix_agent/tools/im_router.ex:452`). Ein fehlgeschlagener Send lässt den Turn offen (`docs/tools-integrations.md:159`), das Modell kann also neu formulieren.
- Vorbild für eine Entscheidung vor dem Turn: `MiniskillSelector` (`internal_session_actor.ex:2343`), Ergebnis als `turn:`-Flag in `TurnReminders.lean`.

---

## 1. Antwortform-Entscheidung vor dem Turn

### Optionen
| Form | Bedeutung | Typischer Fall |
|---|---|---|
| `still` | Keine Nachricht, keine Reaktion. Turn endet leer. | "ok" nach abgeschlossenem Austausch, ein Daumen des Nutzers auf eine Info |
| `reaktion` | Nur eine Reaktion auf die Nutzernachricht, kein Text. | "perfekt, danke!", "trag das ein" (nach Erledigung ✅), "bin morgen weg" (👍) |
| `ein_satz` | Eine Nachricht, ein Satz. | Faktenfrage, Ja/Nein mit Grund, Bestätigung mit Detail |
| `kurz` | 1 bis 3 Nachrichten, je 1 bis 3 Zeilen. | Frage mit zwei Aspekten, kurze Empfehlung |
| `ausfuehrlich` | So lang wie der Inhalt verlangt, aufgeteilt nach Themen. | Vergleich, Entwurf, Recherche-Ergebnis, Plan, wenn der Nutzer Details will |

Zusätzlich zwei unabhängige Fragen im selben Call:
- `reaktion_jetzt`: Soll der Assistent sofort reagieren (ja/nein)? Unabhängig von der Form. Beispiel: Suchauftrag bekommt sofort 🔍, die Antwort kommt später.
- `delegation`: `selbst` oder `helfer` (Details in `08_delegation_und_worker.md`).

### Modell und Aufruf
- Primär: Jev 1.13 über OpenRouter, eine Anfrage mit drei Fragen (Choice über 5 Formen, Noul für `reaktion_jetzt`, Choice für `delegation`). Schatten: DeepSeek V4.1 Flash mit Logprobs, Luna Decisions, Clef-flash (`14_entscheidungsmodelle.md`).
- Eingabe, höchstens 8 KiB:
  - die neue(n) Nutzernachricht(en) dieses Turns mit Zeitstempel,
  - die letzten 6 Nachrichten (Nutzer und Assistent) in Kurzform,
  - Zustand: läuft ein Helfer zu diesem Thema, wartet eine Freigabe, ist die Nachricht eine Antwort auf eine Frage des Assistenten,
  - das gelernte Kommunikationsprofil des Nutzers (kurze Liste aus dem Gedächtnis, Kategorie `kommunikation`, siehe `04`).
- Läuft parallel zum Gedächtnisabruf des Hidden Helpers (`05`). Beide zusammen dürfen höchstens 1,5 s vor dem Turn kosten. Timeout 1,5 s, danach ohne Flag weiter.
- Bei Fehler oder Timeout: kein Flag, der Router entscheidet allein nach den Regeln im Prompt. Nie blockieren.
- Bei Ereignis-Turns (Mail, Erinnerung, Helfer-Ergebnis) läuft dieselbe Entscheidung mit dem Ereignis als Eingabe. Dort ist `still` häufig.

### Übergabe an den Router
- Das Ergebnis wird als `turn:`-Zeile im abschließenden Reminder übergeben. Dieser Teil liegt hinter den Cache-Markern (`VK/Provider/Request.lean:26-49`), bricht also den Cache nicht.
- Format (Beispiel): `turn: form=kurz (0.71, sonst ein_satz 0.22); reaktion_jetzt=ja (0.83); delegation=selbst (0.64)`.
- Ist die Top-Option unter 0,5, steht `unsicher` davor. Dann entscheidet der Router selbst zwischen den beiden besten Optionen.
- Der Router darf abweichen, wenn der Inhalt es verlangt. Die Regel im Prompt: "Die Form ist eine Richtung, keine Grenze. Weiche ab, wenn sonst Wichtiges fehlt oder du etwas Unnötiges sagen würdest."

### Änderungen im Kern
- Vor-Turn-Entscheidung: neuer `TurnFormSelector` nach dem Muster `MiniskillSelector`. Dieser startet in `resolve_round_config/3` (`sa/salix_agent/lib/salix_agent/internal_session_actor.ex:2331-2374`) als `DependencyJob` mit 1 s Frist parallel zum Round-Config-Build, `finish` joint, das Ergebnis wird vor der Runde als Event in die Revision geschrieben (wie `miniskills_selected`, `:2376-2385`; Reducer `VK/Session/Kernel.lean:27`). Für die Antwortform ein neues Event `turn_form_selected` mit Reducer, Write-Validation und Aufnahme in die Reducer-Listen der Beweise (`systems/native/verified_kernel/proofs/VerifiedKernelProofs/Session/AppendOnly/Core.lean:286-287`, `WorkNonRetiring.lean`), damit die Theoreme weiter gelten.
- `turn:`-Flags baut `Request.turnMarker` (`VK/Session/Request.lean:151-164`). Neue Flags `form=…`, `reaktion_jetzt=…`, `delegation=…` mit Wahrscheinlichkeiten. Das Flag `opening=on` (`:142-149`) entfällt.
- `VK/Session/TurnReminders.lean`: Text `opening` (`:16`) streichen. Im statischen Katalog `## Turn reminders` (`:22-28`) die Regeln für die neuen Flags ergänzen: "Die Form ist eine Richtung, keine Grenze. Weiche ab, wenn sonst Wichtiges fehlt oder du Unnötiges sagen würdest. Eine Reaktion ist eine vollständige Antwort. Schweigen ist erlaubt." Ohne Flag: "Antworte in der kleinsten Form, die alles Wichtige enthält."
- Eine Katalogänderung ändert den Prompt-Snapshot und baut den Cache einmal pro Session neu auf (`VK/Session/Query/Reply.lean:398-405`). Das ist einmalig und unkritisch.
- Kernel-Tests: `systems/native/verified_kernel/test/` (Request-Bau, Flags), `systems/apps/salix_agent/test/miniskill_selector_test.exs` als Vorlage für `turn_form_selector_test.exs`. `systems/native/verified_kernel/scripts/check.sh` muss grün sein.
- `resources/salix-system-files/skills/proactive/SKILL.md`: Abschnitt "Write the message" auf dieselben Formen umstellen. Die Pflicht zur abschließenden Frage streichen (Report V19). Reine Information endet ohne Frage.

---

## 2. Sende-Tool

### Verhalten
- Werkzeug bleibt `im_api.internal.send_message`. Jeder Aufruf ist genau eine Chat-Blase und wird sofort zugestellt. Mehrere Aufrufe pro Turn sind erwünscht (E45).
- Die Tool-Beschreibung (Englisch, weil intern) enthält diese Regeln, damit sie bei jedem Aufruf frisch im Blick sind:
  - Eine Nachricht, ein Gedanke oder ein Thema. Getrennte Themen in getrennte Nachrichten.
  - Normal sind 1 bis 3 Zeilen. Fließtext, keine Überschriften, keine Aufzählungen, außer der Inhalt ist eine Liste, die der Nutzer will.
  - Keine Floskeln: kein "Gute Frage", "Gerne!", "Natürlich!", "Ich hoffe, das hilft", keine Wiederholung der Frage, keine Zusammenfassung am Ende, kein Angebot "Soll ich noch...?", außer es gibt einen echten nächsten Schritt.
  - Höchstens eine Frage pro Antwort. Fragen nur bei echtem Blocker.
  - Wenn eine Bestätigung reicht: Reaktion statt Text.
  - Wenn die Nachricht länger als 400 Zeichen ist, setze `lang: true` und nur dann, wenn der Inhalt es verlangt (Entwurf, Liste, Vergleich, Code, ausdrücklicher Wunsch nach Details).
  - Ein Gedanke darf nicht über zwei Nachrichten mitten im Satz geteilt werden.
- Weiche Prüfung am Engpass `im_router.ex:452`: Ist der Text über 400 Zeichen und `lang` fehlt, schlägt der Aufruf mit einer Modell-Fehlermeldung fehl: "Zu lang für eine Chat-Nachricht. Kürze, teile in mehrere Nachrichten zu je einem Thema, oder setze lang=true, wenn der Inhalt es verlangt." Der Turn bleibt offen, das Modell formuliert neu. Nach zwei Ablehnungen im selben Turn wird die dritte Fassung ohne Prüfung zugestellt.
- Nichts wird je abgeschnitten. Kein `max_tokens`-Trick für Kürze.
- Der Wert 400 ist ein Startwert in der Konfiguration (`reply.soft_limit_chars`). Im Eval justieren (`15`).
- Markdown: Fett und einfache Listen werden gerendert, Überschriften im Chat nicht verwendet.

### Schreibstil (kein KI-Slop, E8)
Diese Regeln stehen im Router-Prompt (Abschnitt "Writing style") und in der Beschreibung von `send_message`:
- Schreib wie ein Mensch im Chat: Du-Form, natürliche Sätze, kurze Absätze.
- Keine Gedankenstriche (— oder –) als Stilmittel. Komma, Punkt oder Doppelpunkt.
- Keine Überschriften, keine fetten Zwischentitel, keine Aufzählungswände. Listen nur, wenn der Inhalt eine Liste ist.
- Keine Einleitungen ("Hier ist…", "Kurz zusammengefasst…"), keine Schlussformeln, keine Wiederholung der Frage.
- Keine übertriebene Begeisterung, keine Entschuldigungsfloskeln.
- Emojis nur als Reaktion, nicht im Text, außer der Nutzer schreibt selbst so.
- Zahlen, Daten und Namen genau, ohne Weichmacher ("in etwa", "möglicherweise"), außer es ist wirklich unsicher. Dann einmal klar sagen, was unsicher ist.
- Die Eval-Suite (`15`) prüft diese Regeln mit einem Richter und mit deterministischen Prüfungen (Gedankenstrich, Überschrift, Floskel-Liste).

### Was entfällt
- Kein Nach-Prüfmodell, das fertige Antworten zurückschickt. Der ursprüngliche Wunsch (E48) ist durch die Vor-Turn-Richtung plus Tool-Beschreibung plus weiche Prüfung ersetzt (E46).

---

## 3. Reaktionen

### Datenmodell
- Eine Reaktion gehört zu einer Nachricht: Ziel-Nachricht, Akteur (`assistant` oder `user`), Emoji, Zeitpunkt.
- Pro Akteur höchstens eine Reaktion pro Nachricht. Die neueste gilt, leeres Emoji entfernt.
- Nachrichten sind append-only (`sa/salix_im/lib/salix_im/conversation_message.ex:18`, `ConversationServer` hat kein Edit, `conversation_server.ex:30-624`). Deshalb ist eine Reaktion eine eigene `app_event`-Nachricht mit `metadata.event_type = "message.reaction"`, `target_message_id`, `actor`, `emoji`. Muster: `message.redelivery` (`sa/salix_im/lib/salix_im/conversation_actor.ex:705-753`).
- Keine neue Domänen-Entität. Ergänze in `docs/architecture/DOMAIN_CONCEPTS.md` beim Eintrag Message: "Reactions are app_event annotations; the latest per actor and target wins."
- Anhängen nur über `ConversationServer.append_group_conversation_agent_message` (Assistent, `conversation_server.ex:374`) bzw. `append_group_conversation_message/3` (Nutzer, `:134-135`).
- Clients: Beide filtern `app_event` per Sperrliste (Web `cl/packages/app/src/components/chat/model/conversationChannel.ts:2313-2327`, iOS `cl/packages/apple-core/Sources/CommaCore/Models.swift:344-348`). `message.reaction` dort aufnehmen und stattdessen als Badge an der Ziel-Blase rendern (wie Tapbacks bei iMessage). Projektion "neueste Reaktion pro Akteur und Ziel" im Client-Modell.
- Nutzer-Reaktion: neue Comma-Route `POST /v1/comma/groups/:group_id/conversations/:conversation_id/messages/:message_id/reaction` (neben `router.ex:3550`). Long-Press bzw. Rechtsklick auf eine Blase öffnet eine Emoji-Auswahl (häufige zuerst).

### Werkzeug
- Neue Operation `internal.react` (`im_api.internal.react`): Ziel-Nachricht (Standard: die letzte Nutzernachricht des Turns) und Emoji (beliebiges einzelnes Emoji). Leeres Emoji entfernt.
- Einbau: Manual in `sa/salix_im/lib/salix_im/provider/manuals.ex` (`internal_manual`, ab `:97`, neben `internal.send_message` `:679`), Dispatch in `SalixIM.Provider.Internal.call/3` vor dem Fallback (`provider/internal.ex:399`), IFC in `@contentless` (`sa/salix_agent/lib/salix_agent/ifc/destination.ex:85-93`, dort stehen schon `slack.add_reaction`, `feishu.add_reaction`, `signal.react`). Nicht in `operation_registry.ex` (das führt nur Slack- und Feishu-Operationen).
- Telegram: neue Operation `telegram.set_reaction` (Bot-API `setMessageReaction`) in `sa/salix_im/lib/salix_im/provider/telegram.ex:32-127`, in die Allowlist für verwaltete Verbindungen (`telegram.ex:233-254`), Manual-Eintrag. Für eingehende Nutzer-Reaktionen `message_reaction` in `allowed_updates` von `setWebhook` aufnehmen (`sa/comma_web/lib/comma_web/telegram_bot/req.ex:66-67`) und im Webhook verarbeiten. Ist die Router-Antwort in Telegram, reagiert `internal.react` dort über `telegram.set_reaction`.
- Telegram erlaubt nur 73 feste Emojis und für Bots eine Reaktion pro Nachricht. Abbildung der festen Fälle: 🔍 wird 👀, ⏳ wird 👨‍💻, ✅ wird 👌, 👍 bleibt, ❤️ wird ❤ (ohne Variation Selector), 😂 wird 🤣, 📅 wird 👌, ✉️ wird 👌. Freie Emojis außerhalb der Liste entfallen in Telegram. Die Liste steht in der Bot-API-Doku (`ReactionTypeEmoji`), zur Prüfung liegt sie auch im aiogram-Quelltext.

### Die 8 festen Fälle (stehen in der Tool-Beschreibung)
| Emoji | Wann |
|---|---|
| 🔍 | Du recherchierst oder suchst etwas für diese Nachricht. Sofort setzen, bevor du suchst. |
| ⏳ | Die Arbeit läuft länger im Hintergrund weiter (Helfer gestartet). |
| ✅ | Auftrag erledigt, kein weiterer Text nötig. |
| 👍 | Verstanden oder zur Kenntnis genommen, es ist nichts zu tun. |
| ❤️ | Dank, Lob oder etwas Nettes vom Nutzer. |
| 😂 | Witz oder lustige Nachricht. |
| 📅 | Termin, Erinnerung oder Kalendereintrag angelegt. |
| ✉️ | Mail oder Nachricht im Namen des Nutzers verschickt (nach Freigabe). |

Freie Wahl darüber hinaus ist erlaubt, wenn sie besser passt (🎉 bei guten Nachrichten, 🙏, 😬). Regeln in der Beschreibung:
- Nicht auf jede Nachricht reagieren. Das Flag `reaktion_jetzt` gibt die Richtung.
- Nicht reagieren und dasselbe zusätzlich in Text sagen.
- Eine Reaktion ist eine vollständige Antwort.

### Automatischer Wechsel
- 🔍 und ⏳ sind Zwischenzustände. Die Laufzeit (nicht das Modell) ersetzt sie:
  - 🔍 wird zu ✅, wenn der Turn, der sie gesetzt hat, erfolgreich endet und das Ergebnis zugestellt ist.
  - 🔍 wird zu ⏳, wenn im selben Turn ein Helfer gestartet wird.
  - ⏳ wird zu ✅, wenn das Ergebnis des Helfers zugestellt ist. Scheitert der Helfer, wird die Reaktion entfernt und der Router meldet sich mit Text.
- Setzt das Modell selbst eine andere Reaktion, gilt seine.
- Umsetzung: Die Laufzeit merkt sich pro Nachricht die Zwischenreaktion und die zugehörige Turn- oder Task-ID. Beim Ende des Turns oder der Task wird über `ConversationActor` getauscht.

### Reaktionen des Nutzers
- Reagiert der Nutzer auf eine Assistenten-Nachricht, entsteht ein kleines Ereignis für den Router ("Nutzer reagierte mit 👍 auf: <erste 80 Zeichen>"). Das Weck-Gate (`07`) entscheidet, ob daraus ein Turn wird. Standard: 👍, ❤️, 😂 wecken nicht, ❓, 👎 und unbekannte Emojis wecken.
- Nutzer-Reaktionen sind ein Label-Signal für die Kalibrierung (`14`): 👎 auf eine lange Antwort zählt als "zu lang".

---

## 4. Lernendes Kommunikationsprofil

- Korrekturen wie "kürzer", "mehr Details", "keine Emojis", "schreib mir das als Liste" werden vom Gedächtnis-Erfasser (`04`) als Fakten der Kategorie `kommunikation` gespeichert, mit Gültigkeit und Herkunft.
- Das Profil (höchstens 15 Einträge, gültige Fassung) fließt in jede Antwortform-Entscheidung ein und steht im festen Kernblock des Router-Kontexts (`03`, Abschnitt Kernfakten).
- Widersprüche löst dieselbe Invalidierung wie bei allen Fakten (neuere Vorliebe ersetzt ältere).

---

## 5. Abnahme

- Die Evals in `15_tests_evals_betrieb.md`, Abschnitt "Antwortverhalten", laufen grün.
- Manuell eine Woche Nutzung: Der Nutzer empfindet die Antworten nicht als Romane und vermisst keine Kerninfo. Messgrößen dazu im Dashboard (`15`): Anteil Reaktion/still/ein_satz/kurz/ausführlich, mittlere Zeichen pro Antwort, Anteil `lang=true`, Ablehnungen der weichen Prüfung, Nutzer-Korrekturen zur Länge.
