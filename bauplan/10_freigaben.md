# 10 Freigaben: hartes Aktions-Gate

Ziel: Der Assistent darf alles vorbereiten, aber nichts Unumkehrbares oder nach außen Wirkendes tun, ohne dass der Nutzer genau diese Aktion freigibt. Technisch erzwungen, nicht per Prompt (E65). Für jeden Weg: Tools, Browser, Desktop, Shell, Connectoren, MCP, native Worker (E66).

Das eigentliche Risiko ist nicht "aus Versehen gekauft", sondern Prompt-Injection: Eine Mail oder Webseite gibt dem Agenten Anweisungen. Das trifft Mails senden, Daten löschen, Formulare abschicken.

---

## Ist-Zustand

- Zentrale Stelle für jeden Agenten-Tool-Call: `SessionToolDispatch.authorize/2` (`session_tool_dispatch.ex:84-131`). Gilt für Runden, Loops, `script.run` und den HTTP-Pfad externer Runtimes. Prüfreihenfolge heute: recommendation, guest, inspector, triage, organization, dann IFC.
- IFC ist ein Vertraulichkeits-Gate, kein Aktions-Gate, und standardmäßig aus (`salix_im/ifc/facts.ex:61`, `ifc/check.ex:39-41`). `browser.*` und `env.computer_use` gelten als öffentlicher Ausgang, `sources: []` passiert (`ifc/destination.ex:18-23,518`).
- Geräte: Freigabe pro Group, danach keine Prüfung pro Aktion (`env_dispatch.ex:589-593`, `connector/salix-connect/main.go:4035-4048`).
- `computer_use_start` ist freiwillig (`async_ops.ex:168-190`). `env_dispatch.ex:212-217` prüft es nicht.
- Ohne Gate: `env.exec`, native Worker mit `bypassPermissions` (`connector/salix-connect/claude_runtime.go:434-435`) und `approvalPolicy: never` / `danger-full-access` (`main.go:5975-5976`), MCP-Tools nur als read/write markiert (`salix_mcp/provider.ex:201-206`).
- Bausteine, die es schon gibt: Capability-Requests mit Compare-and-set und Beleg (`capability_requests.ex:20`, `:380-410`, `:456`, Muster `decide_declassification`), einmaliger Verbrauch wie `IFC.consume_receipt` (`ifc.ex:99`), Unknown-Outcome-Sperre im Browser (`browser_bindings.ex:215`), Freigaben an den Nutzer über Web-UI oder Telegram (`async_ops.ex:165`), automatisches Warten wie bei `permission.request` (`async_ops.ex:168-212`).
- Kein allgemeiner Exactly-once-Beleg für Käufe oder Sendungen (`docs/verification.md:87,178-180`).

---

## 1. Effektklassen

Jedes Tool und jede Operation bekommt eine Effektklasse in seinen Metadaten:

| Klasse | Bedeutung | Gate |
|---|---|---|
| `read` | Liest nur | nie |
| `internal` | Ändert nur eigenen Zustand des Assistenten: Gedächtnis, Board, Zeitpläne, UI, Entwürfe, Tasks | nie |
| `external_reversible` | Wirkt nach außen, ist aber leicht rückgängig: Kalender-Entwurf, Label setzen, Datei in eigenem Ordner | Regeln, dann Modell |
| `external_irreversible` | Senden, Posten, Kaufen, Bezahlen, Löschen externer Daten, Formular absenden, Kündigen, Buchen, Veröffentlichen, Überweisen | immer Freigabe |
| `dynamic` | Allzweck-Tools, deren Wirkung vom Ziel abhängt: Browser-Klick, Taste, Formular füllen, Navigation, Desktop-Klick und -Tastatur, `env.exec`, generische MCP-Tools | Inhaltsprüfung, dann Regeln, dann Modell |

- Unbekannte oder nicht klassifizierte Tools, MCP-Schreibtools und Connector-Operationen ohne Klasse gelten als `external_irreversible` (Fail-closed). `read` nur bei ausdrücklicher Markierung.
- Bekannte Sendeoperationen (Gmail senden, Slack posten, Telegram an Dritte, E-Mail-Antworten über Composio) sind `external_irreversible`.
- Nachrichten an den Nutzer selbst (Home, Telegram an ihn) sind `internal`.

---

## 2. ActionGate

Neuer Schritt `ActionGate` in `SessionToolDispatch.authorize/2`, direkt vor `IFC.Check.authorize` (`session_tool_dispatch.ex:130`). Ablauf pro Tool-Call:

1. Effektklasse bestimmen.
2. Bei `dynamic`: Ziel auflösen (Abschnitt 3).
3. Stehende Erlaubnis prüfen (Abschnitt 5). Trifft sie, weiter ohne Rückfrage (nie bei Käufen und Zahlungen).
4. Feste Regeln (Abschnitt 4). Treffer: Freigabe nötig.
5. Kein Treffer und Klasse ist nicht `read`/`internal`: Entscheidungsmodell `action_gate` (`14`), Clef 27B. Bei kalibriert P(unumkehrbar oder extern) ≥ 0,2: Freigabe nötig. Das Modell kann nur eine Freigabe hinzufügen, nie eine aus den Schritten 1 bis 4 entfernen.
6. Freigabe nötig: `action_approval`-Request erzeugen, Tool-Call wartet (Abschnitt 6).
7. Mit gültigem, unverbrauchtem Beleg: ausführen, Beleg verbrauchen.

Jede Entscheidung landet im `decision_log` (`14`) und in einem eigenen Freigabe-Protokoll.

---

## 3. Ziel auflösen bei `dynamic`

### Browser (`systems/apps/salix_agent/lib/salix_agent/browser/commands.ex`, ab Zeile 106: click, fill, press)
- Klick per Ref: Element aus der Ref-Tabelle. Klick per Koordinaten: `document.elementFromPoint(x, y)` über CDP, dann nächster Vorfahr vom Typ button, a, input, `[role=button]`, `[role=link]`.
- Gesammelt: Tag, Rolle und zugänglicher Name (CDP `Accessibility.getPartialAXTree` für den Knoten), sichtbarer Text, `type`, Formular mit `action` und `method`, Ziel-URL bei Links, aktuelle URL, Seitentitel, sichtbare Preisangaben im umgebenden Formular oder Warenkorb.
- `press` mit Enter in einem Formular: wie Klick auf dessen Submit.
- `navigate`: Ziel-URL gegen Zahlungs- und Checkout-Muster.
- `fill`: nur Gate, wenn das Feld ein Zahlungs- oder Passwortfeld auf fremder Domain ist (Schutz vor Exfiltration).

### Desktop (Windows-Dienst `11` und macOS-Helfer)
- Zweistufiges Protokoll: Comma fragt den Dienst zuerst `describe(x, y)` oder `describe(key)`. Der Dienst macht einen UI-Automation-Hit-Test (Windows) bzw. Accessibility-Hit-Test (macOS) und liefert Name, Steuerelementtyp, Automation-ID, Fenstertitel, Prozessname.
- Das Gate entscheidet. Dann ruft Comma `click(x, y, token)`. Der Dienst führt nur mit gültigem Einmal-Token aus und prüft vor dem Klick, dass am Punkt noch dasselbe Element liegt.
- Tastenkürzel mit Wirkung (Ctrl+Enter in Mail-Programmen, Shift+Entf im Explorer, Enter auf einem fokussierten Senden-Button) werden über das fokussierte Element aufgelöst.

### `env.exec` (Shell auf Geräten)
- Befehl parsen. Regeltreffer bei: Netzwerk-Schreibzugriffen (`curl` mit `-X POST|PUT|PATCH|DELETE`, `-d`, `--data`, `-F`, `--upload-file`; `wget --post-*`; `Invoke-WebRequest`/`Invoke-RestMethod` mit `-Method Post|Put|Delete`), Mail-Versand (`sendmail`, `mutt`, `Send-MailMessage`), `git push`, Paket-Veröffentlichung (`npm publish`, `cargo publish`, `twine upload`), Löschen außerhalb des Arbeitsordners (`rm -rf`, `Remove-Item -Recurse`), `ssh`/`scp` zu Fremdhosts, Pipen aus dem Netz in eine Shell.
- Befehle, die der Parser nicht sicher versteht (Variablen-Expansion, `eval`, base64): Modellprüfung mit niedriger Schwelle.

### MCP und Connectoren
- Nach Metadaten. Generische Tools (`execute`, `run`, `call_api`) gelten als `dynamic`: Argumente werden dem Modell vorgelegt.

### Native Worker (Coding-Agenten über `salix-connect`)
- Die hart verdrahteten Flags werden konfigurierbar (`claude_runtime.go:434-435`, `main.go:5975-5976`).
- Standard in diesem Deployment: native Worker sind auf Geräten mit Nutzer-Logins (Windows-VPS, Rechner des Nutzers) aus.
- Wenn eingeschaltet: Claude-Runtime mit `permissionMode: default` und einem PreToolUse-Hook, der jede Bash-, Write- und Netzwerk-Aktion an das ActionGate von Comma meldet (HTTP an den Server über den Connector-Kanal) und auf die Entscheidung wartet. Codex-Runtime mit `approvalPolicy: on-request`, Freigaben über denselben Weg.

---

## 4. Feste Regeln (Deutsch und Englisch)

- **Verben** im zugänglichen Namen, Text oder Label des Ziels: kaufen, bestellen, zahlungspflichtig, jetzt bezahlen, bezahlen, kostenpflichtig, senden, absenden, abschicken, verschicken, löschen, entfernen, endgültig, kündigen, stornieren, bestätigen, buchen, reservieren, veröffentlichen, posten, teilen, überweisen, abonnieren; buy, order, place order, pay, checkout, purchase, subscribe, send, submit, delete, remove, cancel, confirm, book, publish, post, share, transfer.
- **Formulare:** Submit eines Formulars mit `method=post` auf fremder Domain, außer Suchformulare (Feld `type=search` oder `role=search`).
- **URLs:** Pfade mit `checkout`, `payment`, `pay`, `order`, `buy`, `cart/submit`, `spc`, `/gp/buy`; Domains von Zahlungsdiensten (paypal.com, stripe.com Checkout, klarna.com, adyen, mollie, sofort, giropay, apple pay Web, google pay).
- **Zahlungsfelder:** `autocomplete` mit `cc-*`, Felder namens IBAN, Kartennummer, CVC.
- **Tool-Metadaten:** Klasse `external_irreversible`.
- Die Listen liegen als Daten in einer Datei im Repo (nicht im Prompt), mit Tests.

---

## 5. Freigabe-Erlebnis

### Anzeige
- Im Chat als native Freigabekarte (keine Script-Widget-Karte), auf Desktop, Web und iPhone. Inhalt: was genau passiert, in einem Satz plus Details.
  - Mail: Empfänger, Betreff, voller Text.
  - Browser-Klick: Domain, Button-Beschriftung, Betrag falls erkannt, Bildausschnitt des Elements aus dem letzten Screenshot.
  - Desktop: Fenster, Element, Bildausschnitt.
  - Shell: Befehl.
- Auf dem iPhone als Push mit Aktionsknöpfen (`13`). Telegram mit Inline-Knöpfen als Ausweichweg (bestehender Pfad `async_ops.ex:165`).

### Antworten
- **Einmal erlauben.**
- **Immer erlauben für dieses Ziel** (F17): Ziel ist je nach Art der Empfänger (Mail), die Domain plus Button-Name (Browser), das Tool plus Argumentmuster (MCP), der Befehlsanfang (Shell). Nie verfügbar für Käufe, Zahlungen, Löschen und Kündigen.
- **Ablehnen** (optional mit Grund, der an den Router geht).
- Ohne Antwort nach 30 Minuten: abgelaufen, der Router erfährt es und erinnert höchstens einmal, wenn die Sache wichtig ist.

### Stehende Erlaubnisse
- Gespeichert als Regel-Datensatz: Owner, Ziel-Muster, Zeitpunkt, Herkunft (Freigabe-ID). In den Einstellungen sichtbar und löschbar.
- Der Assistent kann keine stehende Erlaubnis selbst anlegen.
- Prüfe mit `DOMAIN_CONCEPTS.md`, ob ein bestehendes Konzept (Capability-Grant, Device-Authorization) das abbildet. Sonst neuer Eintrag "Standing action permission" mit Owner Group.

---

## 6. Beleg und Ausführung

- Neuer Capability-Request-Typ `action_approval` (`capability_requests.ex:20`), Ablauf wie `decide_declassification`: Compare-and-set beim Entscheiden, nur der Gewinner schreibt den Beleg (`capability_requests.ex:380-410,456`).
- Der Beleg ist gebunden an: Hash aus Tool, kanonischen Argumenten, Session und Ziel-Fingerabdruck (URL plus Element-Beschreibung, oder Empfänger plus Text-Hash). Gültig 15 Minuten. Einmal verbrauchbar wie `IFC.consume_receipt` (`ifc.ex:99`).
- Ändert der Agent nach der Freigabe die Argumente (anderer Empfänger, anderer Text), passt der Beleg nicht mehr: neue Freigabe.
- Ausführung höchstens einmal: freigegeben, reserviert (CAS), ausgeführt, bestätigt oder unbekannt. Nie automatisch wiederholt.
- Unbekannter Ausgang (Timeout, Verbindungsabbruch) (V29): Der Router muss den echten Zustand prüfen, bevor er etwas wiederholt. Mail: Gesendet-Ordner per API. Browser: Seite neu lesen, Bestellbestätigung suchen. Erst danach neue Freigabe oder Meldung an den Nutzer.
- Mails werden über eine Entwurfs-ID versendet (Entwurf anlegen ist `internal`, Versand des Entwurfs ist `external_irreversible`). Die Entwurfs-ID dient als Idempotenzschlüssel.

---

## 7. Käufe

- Zusätzlich zur Freigabe nutzt der Assistent nur virtuelle Karten des Nutzers mit Limit (E67), hinterlegt im Browserprofil des Assistenten auf dem Windows-VPS. Nie die Hauptkarte.
- Käufe fragen immer, auch bei bekannten Shops.

---

## 8. Bekannte Restrisiken (im PR dokumentieren)

- Shell-Befehle, die der Parser nicht versteht, verlassen sich auf das Modell (niedrige Schwelle).
- Ein Egress-Proxy auf Geräteebene ist nicht Teil dieses Plans.
- Seiten, die Aktionen ohne erkennbares Element auslösen (JavaScript-Timer), erkennt das Gate nicht. Gegenmittel: Browser-Profil des Assistenten nutzt nur virtuelle Karten; Logins zu kritischen Diensten mit 2FA.

---

## 9. Abnahme

- Tests für jeden Pfad aus Abschnitt 3 mit Regeltreffer, Modelltreffer, stehender Erlaubnis, Ablauf, Argumentänderung nach Freigabe, doppeltem Verbrauch, unbekanntem Ausgang.
- Eval-Suite "Freigaben" (`15`) grün, inklusive Injection-Fälle.
- Abnahmetest 1 und 10 aus `16`.
