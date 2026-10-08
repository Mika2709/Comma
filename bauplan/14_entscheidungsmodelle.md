# 14 Entscheidungsmodelle

Ziel: Überall, wo eine kleine Entscheidung vor einem teuren Schritt sinnvoll ist, entscheidet ein schnelles Modell mit Wahrscheinlichkeiten. Mehrere Modelle laufen im Schatten mit. Die Entscheidung, welches Modell wo primär ist, trifft eine Kalibrierung auf echten Fällen, nicht ein Bauchgefühl. Der Nutzer labelt nichts von Hand.

Nüchterne Ausgangslage aus der Recherche (Stand 2026-10-08):
- Über 20 Entscheidungsmodelle existieren, fast alle aus den letzten drei Wochen, die meisten im Format von Jev (`/v1/systemone`: Choice mit bis zu 255 Optionen, Score mit 2 bis 10 Stufen, Noul ja/nein).
- Eine Auswertung von 28 Preprints (arXiv 2609.32160) fand keinen Genauigkeitsvorteil gegenüber einem normalen LLM, dessen Label-Logprobs man ausliest. Der Vorteil ist Preis und Latenz.
- Rohe Kalibrierung schwankt (Jev ECE 0,04 bis 0,25). Temperatur- oder isotone Skalierung auf einigen hundert Fällen behebt das meiste.
- Prompt-Injection: Ein angehängter Meinungssatz kippt 12,1 Prozent der Jev-Entscheidungen (JevAdvBench). Mail- und Webtext ist fremdgesteuert.
- Jev ist bei deutschen Zahlen und Datumsangaben schwach (79 bis 88 Prozent gegenüber 97 bis 100 bei Claude). Zeitrechnung macht deshalb immer Code, nie das Modell.

---

## Ist-Zustand in Comma

- Ein Anbieter, fest verdrahtet: `Decide.config` (`systems/apps/salix_agent/lib/salix_agent/decide.ex:233-257`, Provider `"typesafe"`), Metering in `llm.ex:210-263`, Anfrage über `decide/provider.ex:5-48`. Config-Keys `decide.{endpoint,model,api_key}` (`salix_store/config_json.ex:309`).
- Anfrageformat `{state, questions, model}`, Antwort `{model, answers, usage{input_tokens, output_tokens}}`, geprüft durch `Decide.decode/3`. Grenzen: 8 Fragen und 12 KiB pro Call.
- Nutzer: `watch.c` (`resources/salix-system-files/skills/proactive/scripts/watch.c:151-191`, still nur bei `quiet` und `confidence_bp >= 9000`), Erinnerungs-Nachprüfung (`comma_web/home_mail.ex:667-697`), Voice-Profil (`salix_agent/router_decision.ex:32-49`), Miniskill-Auswahl (`miniskill_selector.ex:145`), `decide`-Tool für Skripte und Loops.
- `selfhost/configure.py` setzt keinen `decide`-Key. Ohne Key weckt jedes Watch-Ereignis den Router.

---

## 1. Entscheidungs-Router (neues Modul)

Erweitere `SalixAgent.Decide` zu einem Router über mehrere Anbieter. Keine neue Domänen-Entität: Das ist Infrastruktur, kein Domänenkonzept. Das Protokoll (`decision_log`) ist eine Betriebsprojektion.

### Anbieter-Adapter
Alle Adapter liefern dasselbe normalisierte Ergebnis: pro Frage Wahrscheinlichkeit je Option, Konfidenz, Modell-ID mit Version, Latenz, Kosten.

| Adapter | Für | Wie |
|---|---|---|
| `typesafe` | Jev direkt (bestehend) | unverändert |
| `openrouter_decisions` | Jev 1.13, OpenAI Luna Decisions, Clef (27B), Clef-flash, Liquid d1 | `POST https://openrouter.ai/api/alpha/decisions` mit OpenRouter-Key. Format siehe Abschnitt Endpunkte. |
| `logprob_llm` | DeepSeek V4.1 Flash (Together), jedes Chat-Completions-Modell mit Logprobs | Prompt listet Optionen mit Ein-Token-Labels (A, B, C, ...; für Noul J und N). Antwort nur das Label, `max_tokens` 1, `logprobs` an, `top_logprobs` 20. Softmax über die Label-Logprobs ergibt die Verteilung. Eine Frage pro Request, mehrere Fragen parallel. |

### Konfiguration pro Einsatz
Unter `decide.use_cases.<name>` in der Server-Config:
- `primary`: Adapter plus fest gepinnte Modell-ID (nie `latest`).
- `shadows`: Liste weiterer Adapter plus Modell.
- `timeout_ms`, `max_input_bytes`.
- `bands`: Schwellen für die drei Stufen handeln, nachfragen, eskalieren (je Einsatz anders, siehe unten).
- `fallback`: Verhalten bei Fehler oder Timeout.

### Ablauf eines Calls
1. Eingabe bauen (einsatzspezifisch, siehe Einsätze). Fremdtext (Mails, Webseiten, Dateien) steht in klar markierten Blöcken "UNTRUSTED CONTENT".
2. Primär-Call synchron mit Timeout.
3. Schatten-Calls asynchron unter einem eigenen Task-Supervisor. Sie haben nie eine Wirkung.
4. Rohwahrscheinlichkeiten durch die aktuelle Kalibrierungskarte des Paars (Einsatz, Modell) schicken.
5. Ergebnis in `decision_log` schreiben.
6. Bänder anwenden, Ergebnis zurückgeben.

### Protokoll `decision_log` (Postgres)
Felder: ID, Einsatz, Zeitpunkt, Sprache der Eingabe (de/en), Eingabe (vollständig, 90 Tage Aufbewahrung), Optionen, je Modell Rohwahrscheinlichkeiten, kalibrierte Wahrscheinlichkeiten, gewählte Option, Latenz, Kosten, Bezug (Turn-ID, Nachricht-ID, Task-ID, Quell-Item-ID), Label, Labelquelle, Labelzeitpunkt.

---

## 2. Einsätze

| Einsatz | Frage | Primär | Schatten | Bänder und Ausfallverhalten |
|---|---|---|---|---|
| `wake` (Weck-Gate, `07`) | Choice: `still`, `hintergrund`, `melden` | Jev 1.13 | DeepSeek V4.1 Flash (Logprobs), Luna Decisions, Liquid d1 | `still` nur bei kalibriert P(still) ≥ 0,9, sonst mindestens `hintergrund`. Fehler: `hintergrund` (Router entscheidet). Harte Weck-Regeln übersteuern (siehe unten). |
| `reply_form` (`06`) | Choice über 5 Formen, Noul `reaktion_jetzt` | Jev 1.13 | DeepSeek V4.1 Flash, Clef-flash | Unter 0,5 Top-Wahrscheinlichkeit: Flag `unsicher`. Fehler: kein Flag. |
| `delegation` (`08`) | Choice `selbst`, `helfer` | Jev 1.13 | DeepSeek V4.1 Flash | Nur Richtung im Flag. Fehler: kein Flag. |
| `memory_gate` (`05`) | Noul je Kandidat "Braucht der Router diesen Fakt für diese Nachricht?" | Reranker-Schwelle zuerst, Mittelband Jev 1.13 | DeepSeek V4.1 Flash | Über oberer Reranker-Schwelle: rein. Unter unterer: raus. Dazwischen Jev, P ≥ 0,5 rein. Fehler: Mittelband rein (lieber zu viel als zu wenig). |
| `action_gate` (`10`) | Noul "Löst diese Aktion etwas Unumkehrbares oder Externes aus?" | Feste Regeln zuerst, dann Clef 27B (`cloudflare/clef` über OpenRouter) | Jev 1.13, DeepSeek V4.1 Flash | Regeltreffer: immer Freigabe. Sonst Modell: P ≥ 0,2 Freigabe. Das Modell kann nur hinzufügen, nie wegnehmen. Fehler: Freigabe. |
| `auditor_trigger` (`05`) | Noul "Lohnt sich ein Audit dieses Turns?" | DeepSeek V4.1 Flash | Jev | Fehler: Audit läuft. |
| `miniskill` (bestehend) | wie heute | Jev 1.13 | DeepSeek V4.1 Flash | wie heute |
| `reminder_recheck` (bestehend, `home_mail.ex:667-697`) | wie heute | Jev 1.13 | DeepSeek V4.1 Flash | wie heute |
| `voice_length` (bestehend, `router_decision.ex`) | wie heute | Jev 1.13 | keine | wie heute |
| `watch` (`watch.c` über `decide`) | wie heute | Jev 1.13 | DeepSeek V4.1 Flash | Schwelle 9000 bp bleibt, aber auf kalibrierte Werte |

Strands Decider 2B (lokal) ist nicht eingeplant, weil der VPS keine GPU hat. Kommt eine GPU dazu, ist es ein weiterer Schatten-Adapter (`logprob_llm` gegen einen lokalen llama.cpp- oder vLLM-Server).

### Harte Regeln vor dem Weck-Gate
Diese Fälle wecken immer, unabhängig vom Modell, weil ein manipulierter Text das Modell zum Schweigen bringen könnte:
- Absender ist eine Person mit Beziehung im Gedächtnis (Kategorie `person`, markiert als wichtig) oder hat in den letzten 90 Tagen eine Antwort vom Nutzer bekommen.
- Betreff oder Text enthalten (de/en): Frist, Deadline, Mahnung, Rechnung überfällig, Kündigung, dringend, urgent, Termin abgesagt, Termin verschoben.
- Das Ereignis betrifft einen Termin in den nächsten 48 Stunden.
- Das Ereignis antwortet auf einen Thread, in dem der Assistent im Namen des Nutzers geschrieben hat.

---

## 3. Labels aus echten Ergebnissen (automatisch)

Der Nutzer labelt nichts. Labels entstehen aus Verhalten und aus einem nächtlichen Richter-Lauf mit einem starken Modell (das Hauptmodell, Batch, niedrige Priorität).

| Einsatz | Positives/negatives Signal |
|---|---|
| `wake` | Positiv (hätte wecken müssen), wenn der Router nach dem Wecken etwas Sichtbares tat (Nachricht, Todo, Task) oder der Nutzer innerhalb von 24 h darauf einging. Negativ, wenn der Router `still` beendete. Für `still`-Entscheidungen bewertet der Richter jede Nacht alle Fälle mit harten Signalen und eine Stichprobe von 20 Prozent der übrigen ("hätte der Nutzer das wissen wollen?"). Verpasste Wichtige sind der teure Fehler. |
| `reply_form` | Label = Form, die der Router tatsächlich wählte, korrigiert durch Nutzerreaktion: "kürzer", "zu lang", 👎 auf langer Antwort ergeben eine Stufe kleiner; "mehr Details", "und?", "wie genau?" eine Stufe größer. Richter bewertet 20 Prozent Stichprobe. |
| `delegation` | Wenn der Router selbst arbeitete und dabei mehr als 6 Tool-Calls oder 40k neue Kontext-Tokens verbrauchte: `helfer` wäre richtig. Wenn ein Helfer mit höchstens einem Tool-Call fertig war: `selbst` wäre richtig. |
| `memory_gate` | Positiv, wenn der Router den eingespielten Fakt nutzte (Richter prüft Antwort gegen Fakt) oder der Auditor ihn als fehlend meldete. Negativ, wenn eingespielt und ungenutzt. |
| `action_gate` | Richter mit festen Regeln plus Kontext des Ergebnisses (Seite nach dem Klick, Mail versendet, Bestellung erzeugt). Fehlalarme sind billig, Versäumnisse teuer. |

---

## 4. Kalibrierung und Wechsel

- Nächtlicher Job pro Paar (Einsatz, Modell):
  - Labels der letzten 60 Tage, zufällig halbiert.
  - Isotone Regression auf der einen Hälfte, auf der anderen Hälfte messen: Brier-Score, ECE mit Bins gleicher Masse, Genauigkeit im Handlungsband mit Wilson-Intervall.
  - Getrennt für Deutsch und Englisch.
  - Robustheit: Optionsreihenfolge mischen und einen Meinungssatz anhängen, Flip-Rate messen.
  - Kalibrierungskarte speichern. Der Router verwendet ab dann diese Karte.
- Wöchentlicher Wechsel-Job:
  - Ein Schattenmodell wird primär, wenn es mindestens 300 Labels hat, der Brier-Score auf der Testhälfte mindestens 10 Prozent besser ist und der teure Fehler (verpasstes Wecken, fehlende Freigabe) nicht schlechter ist.
  - Wechsel nur zwischen gepinnten Versionen. Jeder Wechsel steht im Betriebs-Dashboard (`15`) mit Zahlen.
  - Beim Aktions-Gate gibt es keinen automatischen Wechsel weg von den festen Regeln. Nur das Zusatzmodell kann wechseln.
- Neue Modellversionen werden nie automatisch übernommen. Neue Version heißt neuer Schatten-Eintrag.

---

## 5. Endpunkte und Modell-IDs

Stand der Web-Recherche 2026-10-08. Die umsetzende AI prüft jede ID beim Einrichten mit einem echten Call und pinnt die datierte Version, wo es eine gibt.

### OpenRouter Decisions
- `POST https://openrouter.ai/api/alpha/decisions` (nicht `/api/v1/alpha/decisions`, das gibt 404). Header `Authorization: Bearer <OpenRouter-Key>`. Diese Modelle funktionieren nicht über `/chat/completions`.
- Anfrage: `model`, `state` (Text, JSON-Objekt oder Array), `questions` als Objekt `{schlüssel: frage}`. Frage-Typen:
  - `choice` mit `instructions` und `criteria` als Objekt `{option: beschreibung}` (ein Array liefert eine Platzhalter-Antwort),
  - `noul` mit `instructions` und `criteria` `{true: ..., false: ...}`,
  - `score` mit `instructions` und `criteria` als geordnete Liste.
- Antwort: `answers.{schlüssel}` mit `type`, bei `choice` `choice`, `probabilities`, `confidence`; bei `noul` das Feld `noul` (Wahrscheinlichkeit); bei `score` Wert und Verteilung. Das passt zu `Decide.decode/3` (`decide.ex:328-439`). Der Adapter mappt nur Pfad, Header und die Hülle.
- Bekannte Schwäche von Jev: Die zuerst genannte Option wird bevorzugt. Der Adapter mischt die Optionsreihenfolge pro Call zufällig und protokolliert sie.

| Modell | OpenRouter-ID | Preis Eingabe pro 1M | Kontext |
|---|---|---|---|
| Jev 1.13 | `typesafe/jev-1.13` (datiert gesehen: `typesafe/jev-1.13-20260917`) | 0,042 $ | 32k |
| Luna Decisions | `openai/gpt-6-luna-decisions` | 0,10 $ | nicht dokumentiert |
| Clef 27B | `cloudflare/clef` | 0,24 $ | 66k |
| Clef-flash 9B | `cloudflare/clef-flash` | 0,09 $ | 66k |
| Liquid d1 | `liquid/d1` (datiert: `liquid/d1-20260930`) | 0,04 $ | 32k bis 65k |

Ausgabe-Tokens sind bei allen kostenlos.

### Logprob-Adapter für DeepSeek V4.1 Flash
- Together AI: Modell-ID `deepseek-ai/DeepSeek-V4.1-Flash`, 0,30 $ Eingabe, 0,006 $ Cache, 1,20 $ Ausgabe pro 1M. OpenAI-kompatibler Chat-Endpunkt `https://api.together.xyz/v1/chat/completions`.
- Ausweichweg: OpenRouter `deepseek/deepseek-v4.1-flash`.
- Ob Logprobs für dieses Modell geliefert werden, ist nicht belegt. Im Thinking-Modus wirken Logprobs bei DeepSeek eventuell nicht. Der Adapter schaltet Thinking für Entscheidungs-Calls ab (`thinking: {type: disabled}` bzw. `reasoning.enabled: false`) und prüft beim Start mit einem Testcall, ob `top_logprobs` kommen. Kommen keine, wird DeepSeek als Entscheidungs-Schatten deaktiviert und im Dashboard vermerkt. Für Crew und Auditor ist das egal, die brauchen keine Logprobs.

### Datenschutz
- Alle Entscheidungs-Calls laufen über den Entscheidungs-Router. Es gibt keinen Nebenpfad mit anderen Regeln (V72). Fremdtext wird als solcher markiert. Der Voice-Profil-Pfad (`router_decision.ex`) wird ebenfalls über den Router geführt.

## 6. Abnahme

- Alle zehn Einsätze laufen über den Router, protokolliert, mit mindestens einem Schatten.
- `selfhost/configure.py` übernimmt OpenRouter-, Together- und DeepInfra-Keys aus `.env` und schreibt sie in die Config.
- Kalibrierungs- und Wechsel-Job laufen als Oban-Cron, mit Ergebnis im Dashboard.
- Tests: Adapter gegen aufgezeichnete Anbieter-Antworten, Normalisierung der Wahrscheinlichkeiten, Bänder, Ausfallverhalten (Timeout, 5xx, kaputtes JSON), harte Weck-Regeln, "Modell nimmt nie eine Freigabe weg".
