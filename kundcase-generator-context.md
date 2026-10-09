# Kundcase-generator – kontext

## Nyckelbeslut

- **Underlag från konsult:** formulär med fasta fält + fritextruta (blandning av strukturerat och inklistrat).
- **Lösningsnivå:** egen liten webbapp (inte bara prompt + Word/Forms, ingen auto-publicering ännu).
- **Drift:** Azure App Service + Entra ID (Easy Auth). Byggs först för localhost, men driftsfärdigt via miljövariabler.
- **Stack:** Python, FastAPI, Jinja2, SQLAlchemy (SQLite lokalt / PostgreSQL på Azure), Claude API.
- **Kundens roll:** styrd granskning – kommentarer per sektion + 3–5 riktade frågor. Ingen fri redigering.
- **Slutgodkännande:** konsulten. Kunden samtycker till citering i formuläret.
- **Export:** ren text per sektion + SEO-paket. CMS (troligen Wix, ej bekräftat) hanteras senare.
- **Mejl:** konsulten skickar kundlänken själv i v1.

## Referenser

- Befintligt case som stilreferens: https://www.mindcamp.se/kundcase-kia
  - Struktur: sammanfattning, Om kunden, Utmaning, Lösning, Resultat & affärsnytta, två citat, CTA.
  - Svagheter ur GEO-perspektiv: inga siffror, ett kundcitat, ingen FAQ/JSON-LD.

## Nyckelfiler

(fylls i under implementationen)
