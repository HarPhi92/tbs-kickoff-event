# Anmeldung → Google Sheet (Zwischenlösung)

Übergangslösung für beide Launch-Days-Events (22.10. Partnerabend, 23.10. Kick-Off), bis
der TBS-Hub steht (siehe [[project_tbs_hub]]). Schreibt jede Formular-Anmeldung als Zeile
in ein eigenes Google Sheet pro Event, kein Double-Opt-in, kein eigenes Backend –
bewusste Entscheidung für Tempo statt der vollen DSGVO-Linie aus
`docs/landingpage-signup-flow.md` im TBS-Hub-Repo.

**Google Cloud Projekt:** `tbs-kick-off-anmeldung` unter dem TBS-Account
`info@therapiebusinessschool.de` (Sheets API + Drive API aktiviert, OAuth-Consent-Screen
im Testmodus mit `info@therapiebusinessschool.de` als Testnutzer) – wird für beide
Workflows mitgenutzt.

## Kick-Off (23.10., `index.html`)

**Status: live seit 31.08.2026.**

- **Google Sheet:** "TBS Kick-Off Anmeldungen".
- **n8n-Workflow:** "TBS Kick-Off – Anmeldung → Google Sheet", published/aktiv.
- **Production-Webhook-URL:** `https://n8n.therapiebusinessschool.de/webhook/kickoff-signup`
  (in `index.html` bei `data-signup-webhook` eingetragen).
- Kopfzeile Sheet: `timestamp | vorname | nachname | email | telefon | praxis |
  fachbereich | herkunft | personenanzahl | verpflegung | einverstanden | event`
  (Spalte `verpflegung` neu, siehe unten – noch nicht im Live-Sheet/Workflow ergänzt).

### TODO (Stand 12.09.2026, noch nicht live nachgezogen)

Die Datei `kickoff-signup-workflow.json` in diesem Ordner enthält bereits die Zielversion
für zwei offene Punkte – der **Live-Workflow in n8n ist aber noch der alte Stand** (nur
Webhook → Sheets), das muss manuell nachgezogen werden:

1. **Spalte `verpflegung`**: Im Formular wird seit dem Mengenwähler-Umbau ein Klartext-Feld
   `verpflegung` mitgeschickt ("keine Verpflegung" oder "n Person(en)"). Im Live-Sheet fehlt
   die Spalte noch, und im Sheets-Node unter *Values to Send* muss das Feld ergänzt werden
   (**Falle 1 oben beachten**: Expression-Modus, nicht Fixed).
2. **Bestätigungsmail ohne Verpflegung**: Neuer `IF`-Node ("Ohne Verpflegung?", prüft
   `{{ $('Webhook').item.json.body.count }} == 0`) → bei 0 Personen ein neuer `Gmail`-Node
   ("Bestätigungsmail senden") direkt nach dem Sheets-Node. Gleiches Mail-Design wie die
   Stripe-Zahlungsbestätigung (von Marcel geliefert), Inhalt aber ohne Zahlungs-/Beleg-
   Erwähnung, mit "Verpflegung: Ohne Mittagessen" statt Personenanzahl-Zeile. Nutzt die
   bestehende Gmail-Credential "Gmail account" (bereits verbunden, siehe Blocker unten) –
   **bei >0 Personen absichtlich keine Mail von hier**, die kommt stattdessen aus dem
   separaten Stripe-Zahlungs-Workflow, sonst gäbe es doppelte Bestätigungsmails.
   Setup: Workflow neu importieren (überschreibt den Live-Stand, vorher exportieren/
   sichern) oder die zwei Nodes manuell nachbauen, Gmail-Credential zuweisen, testen,
   publizieren.

**Bekannter Blocker:** Der Gmail-Versand ist bei `info@therapiebusinessschool.de` durch ein
fehlendes Google-Workspace-Admin-Recht blockiert (`400 Precondition check failed`, siehe
Eintrag vom 06.–09.09.). Bis das mit Marcel geklärt ist, schlägt auch dieser neue Gmail-Node
fehl – Workflow kann trotzdem schon vorbereitet/importiert werden, nur *Publish* + Live-Test
warten darauf.

## Partnerabend (22.10., `partnerabend.html`)

**Status: live seit 02.09.2026.**

- **Google Sheet:** "TBS Partnerabend Anmeldungen" (eigenes Sheet, nicht dasselbe wie
  Kick-Off – andere Formularfelder: `rolle`/`nachricht` statt `fachbereich`/`herkunft`,
  keine Verpflegungspauschale/Stripe-Feld).
- **n8n-Workflow:** "TBS Partnerabend – Anmeldung → Google Sheet", published/aktiv,
  dupliziert vom Kick-Off-Workflow.
- **Production-Webhook-URL:** `https://n8n.therapiebusinessschool.de/webhook/partnerabend-signup`
  (in `partnerabend.html` bei `data-signup-webhook` eingetragen).
- Kopfzeile Sheet: `timestamp | vorname | nachname | email | telefon | praxis | rolle |
  personenanzahl | nachricht | einverstanden | event`
- Kein Stripe/Zahlungsschritt danach (anders als Kick-Off) – die Bestätigungsmeldung
  erscheint direkt nach dem Fire-and-forget-Request.
- **CORS:** Allowed Origins auf `*` belassen.
- Ende-zu-Ende per `curl` getestet: Testzeilen kamen korrekt ausgewertet im Sheet an,
  danach wieder gelöscht.

### TODO (Stand 12.09.2026, noch nicht live nachgezogen)

`partnerabend-signup-workflow.json` enthält bereits einen neuen `Gmail`-Node
("Bestätigungsmail senden") direkt nach dem Sheets-Node – **der Live-Workflow hat den
noch nicht**. Anders als beim Kick-Off-Workflow ist hier **kein IF-Node** nötig: Partnerabend
geht nie über Stripe, jede Anmeldung bekommt also immer diese Mail. Gleiches Design wie die
Kick-Off-/Stripe-Bestätigungsmail, Inhalt auf die Partnerabend-Felder angepasst (Rolle,
Personenanzahl statt Verpflegung, 22.10./17:00–21:00 Uhr statt 23.10.). Nutzt dieselbe
Gmail-Credential "Gmail account" wie der Kick-Off-Workflow – **derselbe Blocker gilt auch
hier** (siehe Kick-Off-Abschnitt oben, Google-Workspace-Admin-Recht auf `info@` fehlt noch).

## Beide gemeinsam

- **CORS:** Allowed Origins auf `*` belassen (Endpoint nimmt nur Anmeldedaten entgegen,
  liefert keine sensiblen Daten zurück).

## Ursprüngliches Setup (zur Referenz, bereits durchgeführt)

1. **Google Sheet anlegen** mit der jeweiligen Kopfzeile in Zeile 1 (siehe oben).

2. **n8n öffnen:** [n8n.therapiebusinessschool.de](https://n8n.therapiebusinessschool.de),
   mit deinem User einloggen (`phillip@therapiebusinessschool.de`).

3. **Workflow importieren/duplizieren:** Workflows → *Import from File* →
   `kickoff-signup-workflow.json` bzw. `partnerabend-signup-workflow.json` aus diesem
   Ordner auswählen (oder bestehenden Workflow duplizieren und Felder anpassen).

4. **Google-Sheets-Node öffnen** ("In Google Sheet speichern"):
   - Credential neu verbinden (Google-Account, OAuth-Bestätigung im Popup) – dafür war ein
     eigener OAuth2-Client in der Google Cloud Console nötig (Projekt anlegen, Sheets- +
     Drive-API aktivieren, OAuth-Consent-Screen mit Testnutzer, Client-ID/Secret erstellen,
     Redirect-URL `https://n8n.therapiebusinessschool.de/rest/oauth2-credential/callback`).
   - Bei *Document* und *Sheet* das Sheet aus der Liste auswählen.
   - **Falle 1:** Die Werte-Felder (Values to Send) landen nach "Add All Columns" im
     *Fixed*-Modus, nicht im *Expression*-Modus – trotz `{{ }}`-Syntax wird der Text dann
     wörtlich statt ausgewertet gespeichert. Für jedes Feld explizit auf den
     "Expression"-Tab-Umschalter neben dem Feldnamen klicken, bevor man tippt.
   - **Falle 2 (beim Duplizieren/Umbenennen von Spalten):** Nach einem Spalten-Rename im
     Sheet über "⋮" neben *Values to Send* → *Refresh Column List* neu synchronisieren,
     sonst bleibt ein Feld leer oder verschwindet beim nächsten Edit stillschweigend aus
     der Werteliste (kein Fehler, kein Hinweis – nur beim Ausführen/Prüfen der Output-JSON
     fällt das fehlende Feld auf). Nach jeder Änderung an den Werte-Feldern die Node per
     *Execute step* testen und die **Output-JSON** (nicht nur die Vorschau unter dem Feld)
     gegenprüfen, bevor man publiziert.

5. **Webhook-Node öffnen** ("Webhook") → *Options* → *Allowed Origins (CORS)* prüfen
   (Standard ist bereits `*`), *Path* auf einen sprechenden Namen setzen
   (`kickoff-signup` / `partnerabend-signup`).

6. **Workflow aktivieren/veröffentlichen** (Publish-Button oben rechts).

7. **Production-Webhook-URL kopieren** (im Webhook-Node oben, Tab *Production URL*).

8. **Im jeweiligen HTML einsetzen:** im `<form>`-Tag bei `data-signup-webhook`.

9. **Testen:** per `curl -X POST <URL> -H "Content-Type: application/json" -d '{...}'`
   prüfen, ob eine neue Zeile im Sheet mit echten (nicht `{{ }}`-literal) Werten in **allen**
   Spalten ankommt – danach Testzeile wieder löschen.

## Bekannte Grenzen dieser Zwischenlösung

- Kein Double-Opt-in, keine Bestätigungsmail.
- Fire-and-forget: schlägt der Webhook fehl, merkt das niemand automatisch – ab und
  zu manuell im Sheet nachsehen, ob neue Anmeldungen ankommen.
- Nach den Launch Days können die Workflows deaktiviert/gelöscht werden, sobald der
  TBS-Hub die echte Lösung übernimmt.
