# Kundcase-generator – plan (spec)

Datum: 2026-10-09
Status: Design godkänd, spec väntar på granskning

## Syfte

Mindcamps kundcase på mindcamp.se ska bli mer synliga i AI-sök (ChatGPT, Perplexity, Google AI Overviews). AI-motorer citerar helst innehåll med konkreta, verifierbara påståenden, namngivna personer och kundens egna ord. Dagens case (t.ex. Kia) har bra struktur men saknar siffror, har få kundcitat och ingen FAQ/strukturerad data.

Lösningen är en intern webbapp som standardiserar flödet: konsult → AI-utkast → kund fyller på med siffror och citat → konsult färdigställer → export till hemsidan.

## Framgångskriterier

- En konsult kan gå från anteckningar till färdigt case utan att skriva från noll.
- Varje case innehåller minst ett kundcitat med namn och titel och, där det finns, minst ett mätbart resultat.
- Varje case levereras med ett SEO-paket (meta, FAQ, JSON-LD).
- Samma kod körs lokalt och på Azure App Service; skillnaden ligger enbart i miljövariabler.

## Flöde och statusar

1. **Utkast** – Konsult loggar in och fyller i formulär: kund, kontaktperson (namn, e-post), utmaning, lösning, teknik, resultat samt fritext (projektbeskrivning, mejl, anteckningar). Formulärdatan sparas. Claude skriver utkast enligt mallen och genererar 3–5 riktade frågor utifrån det som saknas (siffror, citat, namn/titel).
2. **Hos kund** – Konsult granskar och redigerar utkast + frågor, klickar "Skicka till kund". Appen skapar en unik token-länk som konsulten själv mejlar till kunden.
3. **Kund svarat** – Kunden öppnar länken (ingen inloggning), läser caset, kommenterar per sektion och besvarar frågorna. Sidan visar tydligt: "Dina svar kan komma att citeras i det publicerade caset med namn och titel." Kunden kan skicka in en gång; därefter är sidan skrivskyddad.
4. **Slutredigering** – Konsult ser att kunden svarat. Claude väver in svar och kommentarer; konsulten finputsar.
5. **Klar** – Konsult exporterar text per sektion + SEO-paket och lägger in manuellt på hemsidan. Kundlänken stängs.

Konsulten godkänner slutversionen. Kunden gör inget separat slutgodkännande (medvetet val; samtycke till citering ges i steg 3).

## Caset – innehållsmall

- Kort sammanfattning (2–3 meningar, fristående och citerbar för AI)
- Om kunden
- Utmaning
- Lösning
- Resultat & affärsnytta (helst med siffror)
- Kundcitat med namn och titel (minst ett), ev. Mindcamp-citat
- FAQ (3–4 frågor och svar)

Tonen ska följa befintliga case på mindcamp.se (ex. kundcase-kia, ca 600–700 ord).

## Export

- Ren text per sektion (för inklistring i CMS)
- SEO-paket: metatitel, metabeskrivning, FAQ, JSON-LD (schema.org: Article + FAQPage, med kund som `about`/`mentions` och Mindcamp som `publisher`)

CMS-anpassning (Wix/WordPress, HTML-block, automatisk publicering) görs senare.

## Arkitektur

- **FastAPI + Jinja2** – serverrenderade sidor, ingen separat frontend.
- **SQLAlchemy** med `DATABASE_URL`: SQLite lokalt, Azure Database for PostgreSQL (Burstable) i drift.
- **Claude via Microsoft Foundry** (resurs `odmanfoundry`, driftsättning `claude-sonnet-5-5`) för utkast, frågor och invävning.
- **Drift:** Azure App Service (Linux, Python), region Sweden Central. Start via gunicorn med uvicorn-worker.

### Inloggning

- Konsultsidor (`/admin/...`) kräver inloggning. På Azure sköts det av App Service Easy Auth mot Entra ID; appen läser användaren från headern `X-MS-CLIENT-PRINCIPAL-NAME`.
- Lokalt: om `DEV_USER` är satt används den som inloggad användare. `DEV_USER` sätts aldrig på Azure.
- Kundsida (`/c/<token>`) är öppen; token är en lång slumpad sträng (`secrets.token_urlsafe(32)`). Stängd eller okänd token → sidan "Länken är inte längre aktiv".
- Easy Auth konfigureras att tillåta oautentiserade anrop; skyddet av `/admin` görs i appen via headern.

### Moduler

| Fil | Ansvar |
|---|---|
| `app/main.py` | Skapar FastAPI-appen, kopplar routes |
| `app/config.py` | Läser miljövariabler (`.env` lokalt) |
| `app/db.py` | SQLAlchemy-modell `Case` och session |
| `app/auth.py` | Hämtar inloggad konsult (header eller `DEV_USER`) |
| `app/ai.py` | `skriv_utkast()`, `skapa_fragor()`, `vav_in_svar()` |
| `app/prompts/*.txt` | Promptar som textfiler, justerbara utan kodändring |
| `app/routes_admin.py` | Lista, nytt case, redigera, skicka, slutredigera, export |
| `app/routes_kund.py` | Kundsida och inskick av svar |
| `app/export.py` | Bygger text per sektion och SEO-paket |
| `app/templates/` | Jinja2-mallar |

### Datamodell (`cases`)

`id`, `status` (utkast/hos_kund/kund_svarat/klar), `skapad_av`, `kundnamn`, `kontakt_namn`, `kontakt_epost`, `formular` (JSON), `utkast` (JSON per sektion), `fragor` (JSON), `kundsvar` (JSON: svar + kommentarer per sektion), `token`, `token_aktiv`, `skapad`, `uppdaterad`, `kund_svarade`.

## Konfiguration

| Variabel | Lokalt | Azure |
|---|---|---|
| `DATABASE_URL` | `sqlite:///./kundcase.db` | PostgreSQL-anslutningssträng |
| `FOUNDRY_RESOURCE` | `odmanfoundry` | `odmanfoundry` |
| `FOUNDRY_API_KEY` | `.env` | App Settings |
| `DEV_USER` | `jakob.odman@mindcamp.se` | ej satt |
| `BASE_URL` | `http://localhost:8000` | appens publika URL |

## Felhantering

- Formulärdata sparas före varje AI-anrop. Misslyckat Claude-anrop → felmeddelande + "Försök igen", inget går förlorat.
- Ogiltig eller stängd kundlänk → 404-sida "Länken är inte längre aktiv".
- Kund kan skicka in en gång; andra försöket avvisas.

## Test

- pytest för statusövergångar och flöde, med AI-anrop mockade.
- Kundsida: ogiltig token → 404, dubbelinskick avvisas.
- Admin: anrop utan användare (ingen header, ingen `DEV_USER`) → 401.
- Manuell genomkörning med ett riktigt case.

## Utanför första versionen

Automatiska mejl, publicering till CMS, bilder, flera kundkontakter per case, kundens slutgodkännande.
