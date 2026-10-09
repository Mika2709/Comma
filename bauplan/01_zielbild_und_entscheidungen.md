# 01 Zielbild und verbindliche Entscheidungen

## Zielbild in einem Absatz

Ein einziger, endloser Chat mit einem Assistenten, der wie ein guter menschlicher Assistent wirkt. Er weiß, was der Nutzer gesagt hat, auch vor Monaten, und merkt, wenn sich etwas geändert hat. Er antwortet so lang wie nötig: oft nur mit einer Reaktion oder einem Satz, manchmal ausführlich, manchmal gar nicht. 99 Prozent passieren im Hintergrund: Helfer arbeiten unsichtbar, Mails wecken ihn, er erledigt Dinge bis zur Freigabe oder bis sie fertig sind. Zwischenfragen stellt er nur bei echten Blockern. Unumkehrbare Aktionen (senden, kaufen, löschen, Formulare abschicken) laufen technisch erzwungen über eine Freigabe. Rechts sieht der Nutzer ein Todo-Board, das der Assistent selbst pflegt. Fragt der Assistent nach Datum oder Uhrzeit, kommt ein Picker. Er hat einen eigenen Windows-Rechner in der Cloud mit dauerhaften Logins und kommt bei Bedarf an den Rechner des Nutzers. Alles ist auf Deutsch. Der Nutzer pflegt nichts von Hand.

## Gewichtung

Schwer und wichtig, hier liegt die Qualität:
1. Gedächtnis: Erfassen, Korrigieren über Zeit, Herkunft, Abruf, Einspielen, Zeit.
2. Endlos-Chat: Rebuild statt rollendem Fenster, cachefreundlich, ohne Qualitätsverlust.
3. Verhalten des Hauptagenten: kurze Antworten, Schweigen, Reaktionen, kein KI-Slop-Stil.
4. Proaktivität: zuverlässig geweckt, still im Hintergrund, meldet Wichtiges immer.
5. Unsichtbare Delegation.
6. Harte Freigaben.

Leicht, nur Puzzleteile: VPS, Handy, Computersteuerung, Panels. Nicht überbewerten.

---

## Entscheidungen nach Thema

Kennungen `E..` verweisen auf die Entscheidungsliste aus dem Gespräch. Jede Entscheidung ist verbindlich.

### Basis
- Basis ist Comma, selbst gehostet (E1, E2). Kein Wechsel zu Hivekeep, Rakazo, vellum, openbot, OpenInstinct, Grok Bot.
- Teile aus anderen Projekten werden übernommen und angepasst (E4, E5): Hindsight (Gedächtnis), Graphiti (Gültigkeit, Prompts), Mastra Observational Memory (Rebuild-Zeitpunkt, Herkunftsbereiche), vellum v3 (cachefreundliches Einspielen), OpenInstinct (Sende-/Reaktions-Tools, Antwortregeln, Evals), openbot (gestufte Freigaberegeln, Helfer-Relay), Rakazo (gespeicherte exakte Aktionsfreigaben, Zeitstempel im Nutzer-Turn), Hivekeep (`read_message` mit Umgebung, "Offene Punkte"-Abschnitt).
- Solo-Nutzer. Keine Team-Funktionen (E14).
- Elixir und Lean bleiben. Keine Übersetzung nach TypeScript. Der Lean-Kern wird nur dort geändert, wo Rebuild und Auslöser es verlangen (`03`).

### Endlos-Chat und Rebuild (`03`)
- Ein einziger Chat, nie ein neuer (E15). `clear` wird für den Nutzer nicht angeboten.
- Kein rollendes Fenster (E29).
- Auslöser: nur (a) Cache-Ablauf des Hauptmodells und (b) Notbremse kurz unter dem Modell-Maximum. Kein zusätzlicher Größen-Auslöser (E31).
- Unter 100.000 Tokens Kontext kein Rebuild, auch wenn der Cache abläuft (E32). Schwelle ist konfigurierbar.
- Cache-Dauer kommt pro Anbieter und Modell aus der Konfiguration (E33).
- Rebuild statt bloßer Zusammenfassung: Offenes bleibt wörtlich, Erledigtes wird verdichtet, Tool-Ausgaben schrumpfen auf eine Zeile (E30, E36).
- Cleanup-Crew aus mindestens 6 parallelen Calls, jeder mit vollem Kontext und fester Aufgabe (E36 bis E38). Bei zu großem Kontext Aufteilung nach Typ: nur Text, Text mit Thinking, Tool-Calls (E41).
- Zwei Verifier mit unterschiedlichen Modellen prüfen alles und lösen Probleme automatisch (E39).
- Keine AI schreibt direkt in Gedächtnis oder Summary. Worker liefern strukturierte Ergebnisse, normaler Code trägt sie ein (E40).
- Die Summary wächst nicht von Tag zu Tag. Ein Crew-Mitglied hält ein festes Budget ein. Der Kernblock mit den wichtigsten Fakten über den Nutzer bleibt fest im Kontext (E34, `03` Abschnitt 2).
- Wartet der Router auf einen Helfer, wird der Cache warm gehalten, anbieterneutral (E35).

### Gedächtnis (`04`, `05`)
- Gedächtnis fühlt sich nativ an und wird nie manuell gepflegt (E16, E18).
- Automatische Erfassung im Hintergrund nach jedem Turn und beim Rebuild (E17, E36).
- Jeder Fakt kennt seine Originalnachricht. Der Agent kann unsichtbar zur Stelle springen und sie im Zusammenhang lesen (E20).
- Nie zwei widersprüchliche Fakten ohne erkennbare Reihenfolge (E21).
- Veraltete reine Status-Infos ("Paket unterwegs") werden gelöscht. Geänderte Vorlieben und Entscheidungen werden als "ersetzt" markiert und bleiben (E22).
- Zeit ist Kernkonzept: Datum und Uhrzeit in jedem Turn, Zeit stark gewichtet beim Abruf, sehr frische Fakten (1 bis 2 Tage) großzügiger eingespielt, ohne den Rest künstlich zu verschlechtern (E23, E24, E83).
- Verbindungen zwischen Fakten (Person, Zeit, Ursache, Thema) sind zentral (E19).
- Ein Embedder: Qwen3-Embedding. Ein Reranker: Qwen3-Reranker (E26).
- Gedächtnisdienst: geforkter Hindsight als eigener Dienst. Graphiti wird nicht betrieben, nur Gültigkeitsfelder und Prompt-Logik werden übernommen (Claude-Entscheidung, beim Start bestätigen).
- Themenseiten wie bei vellum: die Crew sortiert Wissen in Themen (E42).
- Hidden Helper: vor jedem Turn Suche plus Reranker plus Entscheidungsmodell, angehängter Block; nach jedem Turn Auditor, der Fehlendes nachliefert. Kein Dauer-Mitleser, der dem Router schreibt (E27, E28).

### Antwortform und Reaktionen (`06`)
- Oft nur eine Reaktion oder ein Satz (E43).
- Text geht nur über ein Sende-Tool (E44). Mehrere Nachrichten pro Antwort sind erwünscht (E45).
- Die Tool-Beschreibung erzeugt den "zweiten Gedanken". Ein Entscheidungsmodell gibt vor dem Turn eine grobe Längen-Richtung. Nichts wird abgeschnitten. Kein Token-Limit (E46, E47).
- Reaktionen: freie Emoji-Wahl plus 8 feste Fälle, z.B. Lupe beim Suchen, danach Haken. Nicht auf jede Nachricht, nicht immer Emoji. Wie oft, entscheidet das Entscheidungsmodell (E49, E50).
- Die heutige Pflicht, auf jede Nutzernachricht einen Satz zu senden (`opening`-Regel), fällt weg.

### Proaktivität (`07`)
- Mails und andere Ereignisse wecken den Assistenten (E51). Ein Entscheidungsmodell filtert vorher (E52).
- 99 Prozent im Hintergrund, 100 Prozent abgebbar (E53). Arbeiten bis Freigabe oder fertig, Zwischenfragen nur bei Blockern (E54).
- Beide Budgets (12 Übergaben, 5 Meldungen pro 24 h) fallen ersatzlos weg (E57).
- Bug V39 (Proaktivität bleibt still tot) wird behoben (E58). V50 und V25 ebenfalls.
- Benachrichtigungen kommen zuverlässig an (E56).

### Delegation (`08`)
- Helfer sind unsichtbar, der Nutzer beaufsichtigt nichts (E60). Task-Karten im Chat fallen weg.
- Menschliche Abnahme standardmäßig aus (Claude-Entscheidung, beim Start bestätigen).
- Delegation schützt den Hauptkontext. Statt der harten Prompt-Regel entscheidet ein Entscheidungsmodell weich (E61).

### Entscheidungsmodelle (`14`)
- Breite Auswahl geprüft (E62). Pro Einsatz ein Primärmodell, Kandidaten laufen im Schatten mit (E63, E64).
- Primär: Weck-Gate Jev 1.13; Antwortform und Delegation Jev 1.13; Gedächtnis-Mittelband Jev 1.13 nach Reranker-Schwelle; Aktions-Gate feste Regeln, dann Clef 27B.
- Schattenkandidaten: DeepSeek V4.1 Flash mit Logprobs (E64), OpenAI Luna Decisions, Clef-flash, Liquid d1.
- Abweichung von der früheren Empfehlung "Strands Decider 2B lokal": Der VPS hat keine GPU, auf der CPU ist das Modell für den Weg vor jeder Antwort zu langsam. Deshalb Jev primär. Strands läuft nur, wenn eine GPU dazukommt.
- Abweichung "Clef 27B selbst gehostet": läuft über OpenRouter (`cloudflare/clef`, 66k Kontext) mit fest gepinnter, datierter Modellversion. Selbst hosten bräuchte in voller Genauigkeit eine GPU mit über 50 GB, in 4-Bit etwa 16 GB (Option GPU-Server in `14` Abschnitt 2). Workers AI scheidet aus, weil es langen Text-State auf etwa 2k Tokens kürzt.
- Modellversionen werden fest gepinnt, nie `latest`.
- Beim Aktions-Gate darf ein Modell eine Rückfrage hinzufügen, nie eine wegnehmen.

### Freigaben (`10`)
- Technisch erzwungen, nicht per Prompt (E65).
- Jeder Klick, jede Formular-Absendung, Enter, Navigation zu Checkout und jede MCP- oder Connector-Aktion wird geprüft (E66).
- Käufe zusätzlich über eigene Karten mit Limit des Nutzers (E67). Das Gate bleibt trotzdem, weil Prompt-Injection Mails, Löschen und Formulare trifft.

### UI (`09`)
- Todo-Board rechts ersetzt die Task-Ansicht. Eine Liste, keine Stufen-Tabs (E68).
- Die Haupt-AI kennt das Board im Kontext und pflegt es über ein unsichtbares Tool (E69).
- Die Vorschläge der linken Leiste wandern als markierte Vorschläge aufs Board. Die linke Leiste entfällt. Einstellungen für die Quellen bleiben erreichbar.
- Persönliche Todos sind lokale CalendarItems vom Typ `Task` (bestehende Entität, JSCalendar-Aufgabe). Das Board ist eine Projektion über Todos, laufende Tasks und Vorschläge.
- Datum- und Uhrzeit-Picker im Chat (E72).
- Eigene UI im Chat bleibt (E73). Wiederverwendbare Vorlagen mit festem Datenformat (E74).
- Weitere Panels wie bei Hark: das Board ist das erste Panel. Weitere anheftbare Panels bauen auf demselben Zustandsmodell auf (E70).

### Computer (`11`)
- Assistent läuft immer an auf VPS, nicht auf dem Laptop (E75, E76).
- Eigener Rechner in der Cloud: Windows-VPS mit echtem Desktop. Kein Linux-Desktop (E79).
- Dauerhafte Logins, kein Wegwerf-Browser (E77).
- Mehrere Steuermethoden: Koordinaten als Hauptweg, Bedienelemente-Baum (UI Automation, Accessibility, DOM) zum Lesen und Prüfen, APIs wo vorhanden (E78).
- Desktop-Steuerung über Windows-MCP (fertiger Server mit Koordinaten-Klicks und UI-Automation-Baum), Browser über CDP direkt aus Comma mit Accessibility-Baum (`11`).
- Zugriff auf den eigenen Rechner des Nutzers: macOS über den Comma-Connector, Windows über Windows-MCP.

### Sprache und Zeit (`12`)
- Software komplett auf Deutsch. Code-Namen englisch. Englische Wörter des Nutzers und übliche englische Begriffe bleiben englisch (E82).
- Datum und Uhrzeit in jedem Turn, unsichtbar (E83).

### Kosten
- Qualität vor Kosten (E84). Unnötige Cache-Misses bei großem Kontext vermeiden (E85).

### Handy und Voice (`13`)
- iPhone-App mit Push für Chat und proaktive Meldungen. Telegram als Ausweichkanal.
- Keine Telefonanrufe. Voice-to-Voice in der App mit GPT-Live (E88).

---

## Bewusst nicht gebaut

- Android-App, WhatsApp-Kanal, mehrere Gmail-Konten, Notion-Tiefe, Todoist-Quellen, Team-Funktionen. Kommen später als Puzzleteile, falls nötig.
- Linux-Desktop für den Agenten.
- Eine sichtbare Gedächtnis-Verwaltung für den Nutzer. Es gibt nur eine Debug-Ansicht für Fehlersuche (`15`).
- Graphiti als eigener Dienst mit Graph-Datenbank.
- Ein Dauer-Aufpasser, der jede Nachricht liest und dem Router schreibt.
- Harte Längen- oder Token-Limits.

---

## Festgehaltene Claude-Entscheidungen (beim Start bestätigen lassen)

`README.md` fragt 1 bis 3 einzeln (Fragen 19 bis 21) und 4 bis 15 gesammelt (Frage 22) ab.

1. Abnahme standardmäßig aus.
2. Budgets ersatzlos weg, nur Schleifenschutz gegen identische Wiederholungen.
3. Hindsight als geforkter Dienst, nicht portiert.
4. Zwei Maschinen: Linux-VPS für den Comma-Server (Docker), Windows-VPS als Rechner des Assistenten. Begründung: Comma läuft als Linux-Docker-Stack; Docker auf einem Windows-VPS braucht WSL2 mit verschachtelter Virtualisierung, die viele Anbieter nicht erlauben.
5. Tailscale als privates Netz.
6. Gmail bleibt über Composio angebunden, aber ohne den Filter `is:important`. Das Weck-Gate entscheidet.
7. Interne System-Prompts bleiben Englisch (bessere Befolgung, einfacher Upstream-Merge). Alles, was der Nutzer sieht, ist Deutsch.
8. Die linke Vorschlagsleiste geht im Board auf.
9. Clef 27B über OpenRouter statt selbst gehostet; Embeddings und Reranker über DeepInfra.
10. Desktop-Steuerung über Windows-MCP statt eigenem Windows-Port des Go-Connectors.
11. Zustellung aufs Handy macht der Server (Push, sonst Telegram), nicht mehr der Router.
12. Die iPhone-App wird in GitHub Actions gebaut und über TestFlight installiert, damit kein Mac nötig ist.
13. Reaktionen sind `app_event`-Annotationen an Nachrichten, weil Nachrichten append-only sind.
14. Jev 1.13 über OpenRouter statt Strands Decider 2B lokal (Weck-Gate, Antwortform, Delegation, Gedächtnis-Gate). Nachrichten- und Mailtexte gehen dafür an OpenRouter. Mit GPU-Server wäre es lokal möglich (`14` Abschnitt 2).
15. Kein eigener Graphiti-Dienst mit Graph-Datenbank. Seine Widerspruchs- und Gültigkeitslogik steckt im Hindsight-Fork (`04`).
