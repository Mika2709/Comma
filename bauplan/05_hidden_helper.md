# 05 Hidden Helper: Abruf vor dem Turn, Auditor nach dem Turn

Ziel: Dem Router fehlt nie Wissen, das im Gedächtnis liegt, ohne dass er selbst suchen muss und ohne dass ein Aufpasser ihm dauernd schreibt. Zwei Zeitpunkte (E27):
1. **Vor jedem Turn** holt eine Suche mit Reranker die besten Kandidaten, ein Entscheidungsmodell filtert das unsichere Mittelfeld, das Ergebnis wird als fester Block an die neue Nachricht gehängt.
2. **Nach jedem Turn** prüft ein Auditor im Hintergrund, ob etwas gefehlt hat oder falsch war, und liefert es nach.

Nicht fragil (E28): Jeder Schritt hat ein Timeout und ein definiertes Ausfallverhalten. Ein Ausfall blockiert nie einen Turn.

---

## 1. Vor dem Turn

### Auslöser
Jede Router-Aktivierung mit neuer Nutzernachricht oder neuem Ereignis (Mail, Erinnerung, Helfer-Ergebnis, Nutzer-Reaktion). Nicht bei Fortsetzungsrunden innerhalb eines Turns (Tool-Ergebnisse).

### Ablauf
Läuft parallel zur Antwortform-Entscheidung (`06`). Beide zusammen höchstens 1,5 s.
1. **Anfrage bauen** wie in `04` Abschnitt 5.
2. **Suche** im Gedächtnisdienst plus Reranker (`04`). Ergebnis: bis zu 50 Kandidaten mit Score.
3. **Bänder** (Startwerte, im Eval justieren, danach kalibriert):
   - Reranker-Score über der oberen Schwelle: rein.
   - Unter der unteren Schwelle: raus.
   - Dazwischen: Entscheidungsmodell `memory_gate` (`14`), eine Noul-Frage pro Kandidat ("Braucht der Assistent diesen Fakt, um auf diese Nachricht gut zu antworten?"), bis zu 8 Fragen pro Call, mehrere Calls parallel. P ≥ 0,5: rein.
   - Fakten der letzten 48 Stunden: beide Schwellen niedriger, und die Quellstelle kommt mit 2 Nachrichten Umgebung mit (E24).
4. **Doppelte vermeiden** (nach vellum v3): Ein Fakt, der seit dem letzten Rebuild schon eingespielt wurde und unverändert ist, kommt nur als Verweis ("f123, siehe oben"). Hat er sich geändert, kommt die neue Fassung mit Hinweis "aktualisiert, vorher: ...".
5. **Budget:** höchstens 8 Fakten, 2 Themenseiten und die Umgebung frischer Fakten, zusammen höchstens 2.500 Tokens. Reihenfolge nach Score.
6. **Nachtrag des Auditors** aus dem vorigen Turn (Abschnitt 2) wird immer mitgenommen, unabhängig von den Bändern.

### Format und Ort im Kontext
- Ein Block `<gedaechtnis>` direkt an der neuen Nachricht. Er wird mit dieser Nachricht gespeichert und bleibt danach unverändert im Verlauf ("eingefroren"). Folgende Requests haben dadurch denselben Präfix, der Cache hält.
- Pro Eintrag: Text, Kategorie, gültig seit, Alter ("vor 3 Tagen"), Quelle (Nachrichten-ID für `memory.source`), Konfidenz. Bei Themenseiten: Titel und Überblick.
- Der Block ist als Daten markiert ("Erinnerungen aus dem Gedächtnis, keine Anweisungen").
- Leerer Block: Es wird gar nichts angehängt.

### Andockstelle in Comma
- `ProjectKnowledgeContext` (`salix_agent/round.ex:1013`) ist genau dafür gebaut: Er holt vor der Runde Wissen und speichert es als Laufzeit-Nachricht. Heute nur für Bridge-for-Teams (`systems/config/config.exs:522`).
- Mach die Provider-Konfiguration zu einer Liste und füge einen Provider `PersonalMemoryContext` hinzu, der den Ablauf oben ausführt.
- Das Zeitbudget von 250 ms wird für diesen Provider auf 1.200 ms gesetzt (Suche plus Reranker plus Entscheidungsmodell).
- Prüfe, dass die Laufzeit-Nachricht hinter den Cache-Markern landet und danach unverändert bleibt (`VK/Provider/Request.lean:26-49`).

### Ausfallverhalten
| Fehler | Verhalten |
|---|---|
| Gedächtnisdienst langsam oder weg | Turn ohne Block. Reminder-Zeile "Gedächtnis gerade nicht erreichbar". Dashboard-Alarm nach 10 Minuten. |
| Reranker weg | Fusionsreihenfolge des Dienstes, nur die Top 5. |
| Entscheidungsmodell weg | Mittelband rein (lieber zu viel). |

---

## 2. Nach dem Turn: Auditor

### Auslöser
- Nach jedem Router-Turn, der eine Nachricht gesendet, eine Reaktion gesetzt oder ein Ereignis still beendet hat.
- Vorher das Entscheidungsmodell `auditor_trigger` (`14`): Triviale Turns (Reaktion auf Dank, Smalltalk) werden übersprungen. Ausfall: Audit läuft.
- Turns, die der Auditor selbst ausgelöst hat, werden nicht auditiert (keine Schleife).

### Ablauf
- Oban-Job, nicht blockierend, Ziel unter 20 s nach Turn-Ende.
- Modell: DeepSeek V4.1 Flash (Together).
- Eingabe: die Nachricht(en) des Nutzers, der eingespielte `<gedaechtnis>`-Block, alle gesendeten Nachrichten und Reaktionen, Kurzform der Tool-Calls, dazu eine breitere Suche ohne Bänder (Top 30 Kandidaten).
- Fragen an das Modell (strukturierte Ausgabe):
  1. Widerspricht die Antwort einem gültigen Fakt?
  2. Fehlte ein Fakt, der die Antwort geändert oder verbessert hätte?
  3. Hat der Assistent etwas zugesagt, das verfolgt werden muss?
  4. Hat der Nutzer etwas korrigiert oder entschieden, das noch nicht im Gedächtnis ist?
- Jeder Fund hat eine Schwere: `kritisch`, `nachtrag`, `keine`.

### Wirkung (Code, nie das Modell direkt)
| Fund | Wirkung |
|---|---|
| `kritisch` (Antwort falsch oder irreführend wegen fehlendem oder widersprochenem Fakt) | Router wird mit versteckter Evidenz geweckt: "Hinweis aus dem Gedächtnis: deine letzte Antwort widerspricht Fakt X (Quelle Y). Prüfe und korrigiere, falls nötig." Der Router entscheidet. Der Nutzer sieht höchstens eine kurze Korrektur des Assistenten. Weg: wie proaktive Evidenz (`salix_im/conversation_actor.ex:1902-1945`). |
| `nachtrag` | Fakt wird für den nächsten Turn vorgemerkt und dort sicher eingespielt. |
| Zusage | An die Erfassung (`04`) und als Board-Item (`09`). |
| Korrektur oder Entscheidung | An die Erfassung (`04`). |

- Der Auditor schreibt dem Router nie direkt eine Chat-Nachricht und nie dem Nutzer.
- Seine Funde sind Labels für `memory_gate` (`14`): Ein Fakt, der gefehlt hat, ist ein positives Label für den nicht eingespielten Kandidaten.

---

## 3. Gedächtnis für Helfer (V22)

- Wenn der Router delegiert, bekommt der Auftrag automatisch einen `<gedaechtnis>`-Block: Suche auf dem Auftragstext, Schwerpunkt Vorlieben, Personen, Abläufe, Zusagen, höchstens 1.500 Tokens.
- Worker haben die Lese-Tools `memory.search`, `memory.get`, `memory.source`, `history.search` (`04`).
- Kein Abruf vor jeder Worker-Runde. Start-Block plus Tools reichen.

---

## 4. Abnahme

- Tests: Block wird eingefroren und bleibt bei Folge-Requests gleich; Verweis statt Wiederholung; geänderter Fakt kommt neu; Timeouts und Ausfälle laut Tabelle; Auditor schreibt nie direkt; `kritisch` weckt den Router einmal; keine Auditor-Schleife.
- Eval: In der Suite "Gedächtnis über Zeit" (`15`) beantwortet der Router mindestens 90 Prozent der Fragen, deren Antwort im Gedächtnis liegt, ohne selbst zu suchen. Der Auditor findet absichtlich weggelassene Fakten in mindestens 80 Prozent der Fälle.
