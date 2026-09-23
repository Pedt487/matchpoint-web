# matchpoint.lol — der Web-Host für App-Links

Dieser Ordner ist die komplette statische Website für `matchpoint.lol`.
Sie hat genau zwei Aufgaben:

1. **Die Verknüpfungsdateien**, mit denen iOS und Android Links wie
   `https://matchpoint.lol/oetv/281529` direkt in der App öffnen
   (`.well-known/apple-app-site-association`, `.well-known/assetlinks.json`).
2. **Landeseiten** für Leute ohne App: `/oetv/<id>` und `/t/<id>` zeigen den
   Weg zur öffentlichen Turnierseite des ÖTV und zu den Stores; `/c/<code>`
   erklärt den Einladungscode.

Warum das nötig ist: die App ist seit langem auf `matchpoint.lol` verdrahtet
(`app.json`: `associatedDomains`, `intentFilters`), aber die Dateien lagen
nirgends. Deshalb konnte der Einladungslink nur als `matchpoint://` verschickt
werden, und den kann WhatsApp nicht antippen (PO 22.09.2026: „dann erstmal
raus damit"). Seit demselben Tag will auch „Turnier teilen" einen Link, den
jeder öffnen kann — mit App in MatchPoint, ohne App beim ÖTV.

## Wo das liegt (seit 23.09.2026 live)

**Host: GitHub Pages**, Repository `github.com/Pedt487/matchpoint-web`
(öffentlich — Pages verweigert auf diesem Konto sonst den Dienst, `422 Your
current plan does not support GitHub Pages for this repository`). Der Inhalt
wird **hier** gepflegt und dorthin gespiegelt:

    rm -rf <arbeitskopie> && cp -r web/. <arbeitskopie>/ && cd <arbeitskopie>
    git add -A && git commit && git push

**DNS bei Porkbun**, und zwar skriptbar: `PORKBUN_API_KEY` und
`PORKBUN_SECRET_KEY` stehen in der Repo-`.env`. Gesetzt sind vier A-Records
auf `185.199.108-111.153` und `www` als CNAME auf `pedt487.github.io`. Der
frühere ALIAS auf die Parkseite (`pixie.porkbun.com`) ist weg. **Die sieben
Mail-Einträge (MX, SPF, DKIM für Resend, DMARC) bleiben unberührt** — wer hier
etwas ändert, prüft das vorher im Probelauf.

Prüfen:

    curl -sI https://matchpoint.lol/.well-known/apple-app-site-association
    curl -sI https://matchpoint.lol/.well-known/assetlinks.json

Apple holt die Datei über sein CDN; bis zu 24 h nach der ersten Installation
der App. Google prüft beim Installieren bzw. Aktualisieren.

### Zwei Eigenheiten von GitHub Pages

- **`_headers` und `_redirects` wirken NICHT.** Das sind Cloudflare- und
  Netlify-Dateien; sie liegen weiter hier, falls der Host wechselt. Statt der
  Rewrites fängt `404.html` die Pfade `/oetv/<id>`, `/t/<id>` und `/c/<code>`
  ab und reicht sie an die Landeseiten weiter. Die Adresse bleibt auf einer
  Pages-404-Seite stehen, deshalb ist die Kennung dort noch lesbar.
- **Die AASA-Datei kommt als `application/octet-stream`**, weil sie keine
  Endung hat und Pages den Typ aus der Endung ableitet. Apple verlangt
  `application/json`. Ob es trotzdem funktioniert, zeigt erst die erste
  Installation auf einem iPhone. Wenn nicht, ist der Ausweg Cloudflare Pages,
  wo `_headers` greift — DNS ist eine Minute Arbeit.

`.nojekyll` muss bleiben: Jekyll überspringt Punktordner, `.well-known` wäre
sonst gar nicht veröffentlicht.

### Die Datenschutzerklärung hat EINE Quelle

`privacy.html` wird **erzeugt**, nicht gepflegt:

    python tools/web_privacy.py

Die Quelle ist `site/privacy.txt` im Supabase-Bucket — dieselbe Adresse, die
die Stores als `privacyPolicyUrl` führen und die `app/about.tsx` verlinkt
(`DATENSCHUTZ_URL`). Wer den Text ändert, ändert ihn dort und lässt das Skript
laufen. Zwei gepflegte Fassungen an zwei Adressen laufen sonst auseinander,
und genau davor warnt der Kommentar in `about.tsx`.

`terms.html` hat diese Quelle nicht, es gibt sie nur hier.

## Was vor dem Live-Gang noch fehlt

- **Play-Fingerabdrücke.** `assetlinks.json` kennt bisher nur den Upload-Key
  der Bucket-APK. Die im **Play Store ausgelieferte** App ist mit Googles
  App-Signing-Key signiert; dessen SHA-256 (klassisch UND post-quantum) steht
  nur in der Play Console unter *Test and release → App integrity → App
  signing*. Firebase gibt ihn nicht her, dort stehen nur SHA-1-Werte (geprüft
  23.09.2026 mit `tools/firebase_sha.mjs`). Beide Werte dort abschreiben und
  in `sha256_cert_fingerprints` ergänzen — sonst öffnen Links auf
  Play-Installationen im Browser statt in der App.
- **Neuer Android-Build.** `app.json` hat seit 22.09.2026 die Pfade `/oetv/`
  und `/t/` in den `intentFilters`; Android liest das aus dem Manifest, also
  wirkt es ab dem nächsten Build (1.64.0). iOS liest die Pfade aus der
  AASA-Datei und braucht keinen Build.
- **Der Link im Teilen-Text.** `data/turnierTeilen.ts` verschickt bis dahin
  nur den ÖTV-Link (`https://www.oetv.at/turniere/<id>`), damit kein toter
  Link unterwegs ist. Sobald die Domain antwortet: `MATCHPOINT_WEB` dort
  einschalten, dann steht der App-Link als zweite Adresse im Text.

## Dateien

| Pfad | Zweck |
|---|---|
| `.well-known/apple-app-site-association` | iOS Universal Links: Team `4MJNJA88A4`, Bundle `app.matchpoint.mobile`, Pfade `/c/*`, `/oetv/*`, `/t/*` |
| `.well-known/assetlinks.json` | Android App Links: Paket `app.matchpoint.mobile`, SHA-256 der Signaturzertifikate |
| `_headers` | JSON-Content-Type für beide Dateien (Cloudflare/Netlify) |
| `_redirects` | `/oetv/*`, `/t/*` → `oetv/index.html`; `/c/*` → `c/index.html` (Rewrite, Status 200) |
| `index.html` | Startseite: was MatchPoint ist, die drei Bezugsquellen |
| `oetv/index.html` | Landeseite Turnier: liest die Kennung aus dem Pfad, führt zum ÖTV und zur App |
| `c/index.html` | Landeseite Einladungscode: zeigt den Code, führt zur App und zu den Stores |

Woher der Fingerabdruck kommt: aus der APK gelesen
(`tools/deploy_apk.py`, Signaturblock v2). Die APK hat **einen** Signierer mit
**einem** Zertifikat, SHA-1 `97:A8:…`, SHA-256 `40:8B:B5:38:…` — derselbe
Schlüssel wie in `credentials/matchpoint-upload.jks`. Team-ID aus
`credentials/ios_dist.pem` (OU des Zertifikats).

**Korrektur vom 23.09.2026:** Hier stand bis dahin ein zweites, angeblich
post-quantum signiertes Zertifikat `51:85:…`, und `assetlinks.json` führte
dessen SHA-256. Beides war ein Lesefehler des damaligen Signatur-Scanners, der
Bloecke im Signaturblock gesucht statt den Aufbau gelesen hat. Das Zertifikat
existiert nicht; der Eintrag ist raus. Der Scanner ist ersetzt.
