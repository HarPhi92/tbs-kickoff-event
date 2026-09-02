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
  fachbereich | herkunft | personenanzahl | einverstanden | event`

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
