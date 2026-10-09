# Kundcase-generator – implementationsplan

> **För agenter:** OBLIGATORISK SUB-SKILL: Använd superpowers:executing-plans (eller superpowers:subagent-driven-development) för att genomföra planen task för task. Stegen använder checkboxar (`- [ ]`) för uppföljning. Jakob vill granska ett block i taget – stanna efter varje task och invänta feedback.

**Mål:** En intern webbapp där en konsult får ett AI-skrivet kundcase, skickar en länk till kunden som svarar på riktade frågor, och sedan exporterar en färdig text med SEO-paket.

**Arkitektur:** FastAPI med serverrenderade Jinja2-sidor. SQLAlchemy mot SQLite lokalt och PostgreSQL på Azure; Claude API med structured outputs för utkast, frågor och invävning. Konsultsidor under `/admin` skyddas av Easy Auth-headern (eller `DEV_USER` lokalt), kundsidan `/c/<token>` är öppen.

**Tech stack:** Python 3.12+, FastAPI, Jinja2, SQLAlchemy 2, python-multipart, python-dotenv, anthropic, psycopg, gunicorn/uvicorn, pytest.

**Spec:** `kundcase-generator-plan.md` (i projektroten). Läs den innan du börjar.

## Globala begränsningar

- All kod körs i projektets `venv/`. Installera aldrig paket globalt.
- Samma kod lokalt och på Azure – skillnaden ligger enbart i miljövariabler (`DATABASE_URL`, `ANTHROPIC_API_KEY`, `DEV_USER`, `BASE_URL`, `CLAUDE_MODEL`).
- Modell: `claude-sonnet-5-5`. Sonnet 5.5 stödjer inte forcerad `tool_choice` – använd `output_config.format` (JSON-schema).
- Statusar: `utkast` → `hos_kund` → `kund_svarat` → `klar`. Andra övergångar ger 409.
- Kundlänkens token: `secrets.token_urlsafe(32)`.
- Kundsidan visar alltid: "Dina svar kan komma att citeras i det publicerade caset med namn och titel."
- AI:n får aldrig hitta på siffror, citat, namn eller titlar. Saknad info markeras som `[VERSALER: ...]`.
- JSON-kolumner ändras genom att tilldela ett nytt objekt (`case.utkast = {...}`), aldrig genom att mutera på plats – annars sparar SQLAlchemy inte ändringen.
- Kodstil: som Jakob skriver själv – svenska namn, få kommentarer, ingen onödig abstraktion.

## Review focus

1. **Konsulten redigerar och klickar "Spara och skicka" / "Spara och markera klar" utan att först klicka "Spara"** → ändringarna ska följa med. Testas i Task 6 (`test_skicka_sparar_andringar_och_skapar_kundlank`) och Task 8 (`test_klar_sparar_stanger_lank_och_visar_export`).
2. **Claude-anropet misslyckas eller avbryts** → formulärdatan finns kvar, felet visas och "Skriv utkast med AI" går att köra igen. Testas i Task 4 och Task 5. På Azure måste gunicorns timeout höjas (Task 10).
3. **Kunden lämnar vissa frågor tomma** → inskicket accepteras och tomma svar lagras som `""`. Testas i Task 7 (`test_kund_skickar_svar`).
4. **Kunden öppnar länken igen efter inskick, eller efter att caset är klart** → skrivskyddad tacksida respektive 404 "Länken är inte längre aktiv". Testas i Task 7 och Task 8.
5. **Markeringar som `[SIFFRA: ...]` finns kvar i slutversionen** → varning på case- och exportsidan. Testas i Task 3 och Task 8.

---

## Filstruktur

```
kundcase-generator/
├── app/
│   ├── __init__.py
│   ├── config.py          # miljövariabler
│   ├── db.py              # Case-modell, engine, session
│   ├── auth.py            # inloggad_konsult()
│   ├── sektioner.py       # SEKTIONER, FAQ-text, markeringar
│   ├── export.py          # bygg_export()
│   ├── ai.py              # skriv_utkast(), skapa_fragor(), vav_in_svar()
│   ├── prompts/           # utkast.txt, fragor.txt, invavning.txt
│   ├── mallar.py          # Jinja2Templates + globals
│   ├── main.py            # FastAPI-app
│   ├── routes_admin.py    # /admin/...
│   ├── routes_kund.py     # /c/<token>
│   └── templates/         # *.html
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── hjalp.py           # testdata
│   └── test_*.py
├── pytest.ini
├── requirements.txt
├── .env.example
└── README.md
```

---

### Task 1: Projektgrund, konfiguration och databas

**Filer:**
- Skapa: `requirements.txt`, `.env.example`, `pytest.ini`, `app/__init__.py`, `app/config.py`, `app/db.py`, `tests/__init__.py`, `tests/conftest.py`, `tests/test_db.py`

**Gränssnitt:**
- Producerar: `config.DATABASE_URL`, `config.DEV_USER`, `config.BASE_URL`, `config.CLAUDE_MODEL`; `db.Case`, `db.init_db(engine)`, `db.get_session()`, `db.nu()`; fixtures `session_fabrik`, `session`.

- [ ] **Steg 1: Skapa projektfiler och venv**

`requirements.txt`:
```
fastapi
uvicorn[standard]
gunicorn
jinja2
python-multipart
sqlalchemy
psycopg[binary]
python-dotenv
anthropic
pytest
httpx
```

`.env.example`:
```
DATABASE_URL=sqlite:///./kundcase.db
ANTHROPIC_API_KEY=
DEV_USER=jakob.odman@mindcamp.se
BASE_URL=http://localhost:8000
CLAUDE_MODEL=claude-sonnet-5-5
```

`pytest.ini`:
```
[pytest]
pythonpath = .
testpaths = tests
```

`app/__init__.py` och `tests/__init__.py`: tomma filer.

Kör:
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

- [ ] **Steg 2: Skriv konfigurationen**

`app/config.py`:
```python
import os

from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./kundcase.db")
DEV_USER = os.getenv("DEV_USER", "")
BASE_URL = os.getenv("BASE_URL", "http://localhost:8000")
CLAUDE_MODEL = os.getenv("CLAUDE_MODEL", "claude-sonnet-5-5")
```
`ANTHROPIC_API_KEY` läses direkt av anthropic-klienten från miljön.

- [ ] **Steg 3: Skriv det fallerande testet**

`tests/conftest.py`:
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

from app import db


@pytest.fixture
def session_fabrik():
    engine = create_engine(
        "sqlite://", connect_args={"check_same_thread": False}, poolclass=StaticPool
    )
    db.init_db(engine)
    return sessionmaker(bind=engine, expire_on_commit=False)


@pytest.fixture
def session(session_fabrik):
    with session_fabrik() as s:
        yield s
```

`tests/test_db.py`:
```python
from app.db import Case


def test_nytt_case_far_standardvarden(session):
    case = Case(skapad_av="test@mindcamp.se", kundnamn="Acme AB")
    session.add(case)
    session.commit()

    assert case.id is not None
    assert case.status == "utkast"
    assert case.utkast == {}
    assert case.fragor == []
    assert case.token is None
    assert case.token_aktiv is False
    assert case.skapad is not None


def test_json_falt_sparas_och_lases(session):
    case = Case(skapad_av="x", kundnamn="Acme AB", utkast={"titel": "Åäö"}, fragor=["F1"])
    session.add(case)
    session.commit()
    session.expire_all()

    hamtad = session.get(Case, case.id)
    assert hamtad.utkast == {"titel": "Åäö"}
    assert hamtad.fragor == ["F1"]
```

- [ ] **Steg 4: Kör testet och se att det fallerar**

Kör: `pytest tests/test_db.py -v`
Förväntat: FAIL med `ModuleNotFoundError: No module named 'app.db'`

- [ ] **Steg 5: Skriv databasmodulen**

`app/db.py`:
```python
from datetime import datetime, timezone

from sqlalchemy import JSON, Boolean, DateTime, String, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker

from app import config


def nu():
    return datetime.now(timezone.utc)


class Base(DeclarativeBase):
    pass


class Case(Base):
    __tablename__ = "cases"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[str] = mapped_column(String(20), default="utkast")
    skapad_av: Mapped[str] = mapped_column(String(200))
    kundnamn: Mapped[str] = mapped_column(String(200))
    kontakt_namn: Mapped[str] = mapped_column(String(200), default="")
    kontakt_epost: Mapped[str] = mapped_column(String(200), default="")
    formular: Mapped[dict] = mapped_column(JSON, default=dict)
    utkast: Mapped[dict] = mapped_column(JSON, default=dict)
    fragor: Mapped[list] = mapped_column(JSON, default=list)
    kundsvar: Mapped[dict] = mapped_column(JSON, default=dict)
    token: Mapped[str | None] = mapped_column(String(100), unique=True, nullable=True)
    token_aktiv: Mapped[bool] = mapped_column(Boolean, default=False)
    skapad: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=nu)
    uppdaterad: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=nu, onupdate=nu)
    kund_svarade: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)


connect_args = {"check_same_thread": False} if config.DATABASE_URL.startswith("sqlite") else {}
engine = create_engine(config.DATABASE_URL, connect_args=connect_args)
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


def init_db(eng=engine):
    Base.metadata.create_all(eng)


def get_session():
    with SessionLocal() as session:
        yield session
```

- [ ] **Steg 6: Kör testerna**

Kör: `pytest tests/test_db.py -v`
Förväntat: 2 passed

- [ ] **Steg 7: Commit**

```bash
git add requirements.txt .env.example pytest.ini app/ tests/
git commit -m "Projektgrund, konfiguration och databasmodell"
```

---

### Task 2: Inloggning

**Filer:**
- Skapa: `app/auth.py`, `tests/test_auth.py`

**Gränssnitt:**
- Konsumerar: `config.DEV_USER`
- Producerar: `inloggad_konsult(request: Request) -> str` – returnerar e-post, kastar `HTTPException(401)`.

- [ ] **Steg 1: Skriv det fallerande testet**

`tests/test_auth.py`:
```python
import pytest
from fastapi import HTTPException
from starlette.requests import Request

from app import config
from app.auth import inloggad_konsult


def _request(headers=None):
    return Request({
        "type": "http",
        "headers": [(k.lower().encode(), v.encode()) for k, v in (headers or {}).items()],
    })


def test_easy_auth_header_anvands(monkeypatch):
    monkeypatch.setattr(config, "DEV_USER", "dev@mindcamp.se")
    request = _request({"X-MS-CLIENT-PRINCIPAL-NAME": "anna@mindcamp.se"})
    assert inloggad_konsult(request) == "anna@mindcamp.se"


def test_dev_user_anvands_lokalt(monkeypatch):
    monkeypatch.setattr(config, "DEV_USER", "dev@mindcamp.se")
    assert inloggad_konsult(_request()) == "dev@mindcamp.se"


def test_ingen_anvandare_ger_401(monkeypatch):
    monkeypatch.setattr(config, "DEV_USER", "")
    with pytest.raises(HTTPException) as fel:
        inloggad_konsult(_request())
    assert fel.value.status_code == 401
```

- [ ] **Steg 2: Kör testet och se att det fallerar**

Kör: `pytest tests/test_auth.py -v`
Förväntat: FAIL med `ModuleNotFoundError: No module named 'app.auth'`

- [ ] **Steg 3: Skriv inloggningen**

`app/auth.py`:
```python
from fastapi import HTTPException, Request

from app import config


def inloggad_konsult(request: Request) -> str:
    # På Azure sätter Easy Auth headern och tar bort den från inkommande anrop
    anvandare = request.headers.get("X-MS-CLIENT-PRINCIPAL-NAME") or config.DEV_USER
    if not anvandare:
        raise HTTPException(status_code=401, detail="Inte inloggad")
    return anvandare
```

- [ ] **Steg 4: Kör testerna**

Kör: `pytest tests/test_auth.py -v`
Förväntat: 3 passed

- [ ] **Steg 5: Commit**

```bash
git add app/auth.py tests/test_auth.py
git commit -m "Inloggning via Easy Auth-header eller DEV_USER"
```

---

### Task 3: Sektioner och export

**Filer:**
- Skapa: `app/sektioner.py`, `app/export.py`, `tests/test_sektioner.py`, `tests/test_export.py`

**Gränssnitt:**
- Producerar:
  - `SEKTIONER: list[tuple[str, str]]` – `(nyckel, rubrik)` i ordning: sammanfattning, om_kunden, utmaning, losning, resultat, citat
  - `faq_till_text(faq: list[dict]) -> str`, `text_till_faq(text: str) -> list[dict]` – FAQ-poster är `{"fraga": str, "svar": str}`
  - `hitta_markeringar(utkast: dict) -> list[str]`
  - `bygg_export(case) -> dict` med nycklarna `titel`, `sektioner` (lista av `(rubrik, text)`), `faq`, `meta_titel`, `meta_beskrivning`, `json_ld` (str), `markeringar`

- [ ] **Steg 1: Skriv de fallerande testerna**

`tests/test_sektioner.py`:
```python
from app.sektioner import faq_till_text, hitta_markeringar, text_till_faq


def test_faq_fram_och_tillbaka():
    faq = [{"fraga": "Vad är det?", "svar": "Ett case."}, {"fraga": "Hur?", "svar": "Så här."}]
    assert text_till_faq(faq_till_text(faq)) == faq


def test_text_till_faq_ignorerar_skrap():
    text = "Hej\nS: svar utan fråga\nF: Fråga?\n\nS: Svar.\nF: Fråga utan svar"
    assert text_till_faq(text) == [{"fraga": "Fråga?", "svar": "Svar."}]


def test_text_till_faq_hanterar_windows_radbrytningar():
    assert text_till_faq("F: A?\r\nS: B.") == [{"fraga": "A?", "svar": "B."}]


def test_hitta_markeringar():
    utkast = {"resultat": "Sparar [SIFFRA: timmar per månad].", "citat": "[KUNDCITAT]", "losning": "Power BI [beta]"}
    assert hitta_markeringar(utkast) == ["[SIFFRA: timmar per månad]", "[KUNDCITAT]"]
```

`tests/test_export.py`:
```python
import json
from datetime import datetime
from types import SimpleNamespace

from app.export import bygg_export


def _case(**utkast):
    standard = {
        "titel": "En skalbar grund för automatisering",
        "sammanfattning": "Mindcamp hjälpte Acme AB att automatisera.",
        "om_kunden": "O", "utmaning": "U", "losning": "L", "resultat": "R", "citat": "C",
        "faq": [{"fraga": "Vad gjorde Mindcamp?", "svar": "Automatiserade flöden."}],
    }
    standard.update(utkast)
    return SimpleNamespace(kundnamn="Acme AB", utkast=standard, uppdaterad=datetime(2026, 10, 9, 12, 0))


def test_export_har_sektioner_i_ordning():
    export = bygg_export(_case())
    assert [rubrik for rubrik, _ in export["sektioner"]] == [
        "Sammanfattning", "Om kunden", "Utmaning", "Lösning", "Resultat & affärsnytta", "Citat",
    ]


def test_json_ld_har_article_och_faq():
    data = json.loads(bygg_export(_case())["json_ld"])
    typer = [n["@type"] for n in data["@graph"]]
    assert typer == ["Article", "FAQPage"]
    artikel = data["@graph"][0]
    assert artikel["publisher"]["name"] == "Mindcamp"
    assert artikel["about"]["name"] == "Acme AB"
    assert artikel["datePublished"] == "2026-10-09"
    assert data["@graph"][1]["mainEntity"][0]["name"] == "Vad gjorde Mindcamp?"


def test_json_ld_utan_faq_saknar_faqpage():
    data = json.loads(bygg_export(_case(faq=[]))["json_ld"])
    assert [n["@type"] for n in data["@graph"]] == ["Article"]


def test_json_ld_kan_inte_bryta_script_tagg():
    export = bygg_export(_case(titel="Test </script><script>alert(1)"))
    assert "</script>" not in export["json_ld"]
    assert json.loads(export["json_ld"])["@graph"][0]["headline"].startswith("Test </script>")


def test_meta_kortas():
    export = bygg_export(_case(titel="Ord " * 30, sammanfattning="Ord " * 80))
    assert export["meta_titel"].endswith(" | Mindcamp")
    assert len(export["meta_titel"]) <= 60
    assert len(export["meta_beskrivning"]) <= 155


def test_export_listar_kvarvarande_markeringar():
    assert bygg_export(_case(resultat="[SIFFRA: tid]"))["markeringar"] == ["[SIFFRA: tid]"]
```

- [ ] **Steg 2: Kör testerna och se att de fallerar**

Kör: `pytest tests/test_sektioner.py tests/test_export.py -v`
Förväntat: FAIL med `ModuleNotFoundError`

- [ ] **Steg 3: Skriv sektionsmodulen**

`app/sektioner.py`:
```python
import re

SEKTIONER = [
    ("sammanfattning", "Sammanfattning"),
    ("om_kunden", "Om kunden"),
    ("utmaning", "Utmaning"),
    ("losning", "Lösning"),
    ("resultat", "Resultat & affärsnytta"),
    ("citat", "Citat"),
]

MARKERING = re.compile(r"\[[A-ZÅÄÖ]{3,}[^\]]*\]")


def faq_till_text(faq):
    return "\n\n".join(f"F: {p['fraga']}\nS: {p['svar']}" for p in faq)


def text_till_faq(text):
    faq = []
    fraga = None
    for rad in text.splitlines():
        rad = rad.strip()
        if rad.startswith("F:"):
            fraga = rad[2:].strip()
        elif rad.startswith("S:") and fraga:
            faq.append({"fraga": fraga, "svar": rad[2:].strip()})
            fraga = None
    return faq


def hitta_markeringar(utkast):
    text = "\n".join(str(utkast.get(nyckel, "")) for nyckel, _ in SEKTIONER)
    return MARKERING.findall(text)
```

- [ ] **Steg 4: Skriv exportmodulen**

`app/export.py`:
```python
import json

from app.sektioner import SEKTIONER, hitta_markeringar

MINDCAMP = {"@type": "Organization", "name": "Mindcamp", "url": "https://www.mindcamp.se"}


def _korta(text, max_langd):
    text = " ".join(text.split())
    if len(text) <= max_langd:
        return text
    return text[: max_langd - 1].rsplit(" ", 1)[0] + "…"


def bygg_export(case):
    utkast = case.utkast
    titel = utkast.get("titel", "")
    sammanfattning = utkast.get("sammanfattning", "")
    faq = utkast.get("faq", [])

    graf = [{
        "@type": "Article",
        "headline": titel,
        "description": sammanfattning,
        "datePublished": case.uppdaterad.date().isoformat(),
        "author": MINDCAMP,
        "publisher": MINDCAMP,
        "about": {"@type": "Organization", "name": case.kundnamn},
    }]
    if faq:
        graf.append({
            "@type": "FAQPage",
            "mainEntity": [
                {"@type": "Question", "name": p["fraga"], "acceptedAnswer": {"@type": "Answer", "text": p["svar"]}}
                for p in faq
            ],
        })
    json_ld = json.dumps({"@context": "https://schema.org", "@graph": graf}, ensure_ascii=False, indent=2)

    return {
        "titel": titel,
        "sektioner": [(rubrik, utkast.get(nyckel, "")) for nyckel, rubrik in SEKTIONER],
        "faq": faq,
        "meta_titel": _korta(titel, 60 - len(" | Mindcamp")) + " | Mindcamp",
        "meta_beskrivning": _korta(sammanfattning, 155),
        "json_ld": json_ld.replace("</", "<\\/"),
        "markeringar": hitta_markeringar(utkast),
    }
```

- [ ] **Steg 5: Kör testerna**

Kör: `pytest tests/test_sektioner.py tests/test_export.py -v`
Förväntat: 10 passed

- [ ] **Steg 6: Commit**

```bash
git add app/sektioner.py app/export.py tests/test_sektioner.py tests/test_export.py
git commit -m "Sektioner, FAQ-hantering och export med SEO-paket"
```

---

### Task 4: AI-modul och promptar

**Filer:**
- Skapa: `app/ai.py`, `app/prompts/utkast.txt`, `app/prompts/fragor.txt`, `app/prompts/invavning.txt`, `tests/test_ai.py`

**Gränssnitt:**
- Konsumerar: `config.CLAUDE_MODEL`, `SEKTIONER`
- Producerar:
  - `class AiFel(Exception)`
  - `skriv_utkast(formular: dict) -> dict` – nycklar: `titel`, alla `SEKTIONER`-nycklar, `faq`
  - `skapa_fragor(formular: dict, utkast: dict) -> list[str]`
  - `vav_in_svar(utkast: dict, fragor: list[str], kundsvar: dict) -> dict` – samma form som `skriv_utkast`

- [ ] **Steg 1: Skriv det fallerande testet**

`tests/test_ai.py`:
```python
import json
from types import SimpleNamespace

import anthropic
import pytest

from app import ai


class FalskKlient:
    def __init__(self, svar=None, fel=None):
        self.svar = svar
        self.fel = fel
        self.anrop = []
        self.beta = self
        self.messages = self

    def create(self, **kwargs):
        self.anrop.append(kwargs)
        if self.fel:
            raise self.fel
        return self.svar


def _svar(data, stop_reason="end_turn"):
    return SimpleNamespace(
        stop_reason=stop_reason,
        content=[SimpleNamespace(type="text", text=json.dumps(data, ensure_ascii=False))],
    )


@pytest.fixture
def klient(monkeypatch):
    def installera(**kwargs):
        falsk = FalskKlient(**kwargs)
        monkeypatch.setattr(ai.anthropic, "Anthropic", lambda: falsk)
        return falsk
    return installera


def test_skriv_utkast_returnerar_json(klient):
    falsk = klient(svar=_svar({"titel": "Titel"}))
    assert ai.skriv_utkast({"kund": "Acme"}) == {"titel": "Titel"}

    anrop = falsk.anrop[0]
    assert anrop["model"] == "claude-sonnet-5-5"
    assert "kundcase" in anrop["system"]
    assert "Acme" in anrop["messages"][0]["content"]
    assert anrop["output_config"]["format"]["type"] == "json_schema"


def test_skapa_fragor_returnerar_lista(klient):
    klient(svar=_svar({"fragor": ["Hur mycket tid sparar ni?"]}))
    assert ai.skapa_fragor({"kund": "Acme"}, {"titel": "T"}) == ["Hur mycket tid sparar ni?"]


def test_vav_in_skickar_kundsvar(klient):
    falsk = klient(svar=_svar({"titel": "Ny"}))
    ai.vav_in_svar({"titel": "T"}, ["F1"], {"namn": "Anna Andersson"})
    assert "Anna Andersson" in falsk.anrop[0]["messages"][0]["content"]


def test_api_fel_blir_aifel(klient):
    klient(fel=anthropic.APIConnectionError(request=None))
    with pytest.raises(ai.AiFel):
        ai.skriv_utkast({"kund": "Acme"})


def test_avbrutet_svar_blir_aifel(klient):
    klient(svar=_svar({}, stop_reason="max_tokens"))
    with pytest.raises(ai.AiFel, match="max_tokens"):
        ai.skriv_utkast({"kund": "Acme"})
```

- [ ] **Steg 2: Kör testet och se att det fallerar**

Kör: `pytest tests/test_ai.py -v`
Förväntat: FAIL med `ImportError: cannot import name 'ai'`

- [ ] **Steg 3: Skriv promptarna**

`app/prompts/utkast.txt`:
```
Du skriver kundcase för Mindcamp, ett svenskt konsultbolag inom data, analys, AI och automatisering. Casen publiceras på mindcamp.se och ska kunna hittas och citeras av både människor och AI-assistenter.

Du får underlag från en konsult som JSON: kund, utmaning, lösning, teknik, resultat och fria anteckningar. Skriv ett kundcase utifrån underlaget.

Ton och längd
- Svenska, sakligt och konkret, utan säljfloskler. Totalt 500–700 ord.
- Skriv i tredje person om både kunden och Mindcamp.

Fälten i svaret
- titel: en rubrik som säger vad kunden uppnådde, t.ex. "En skalbar grund för automatisering och AI med Power Automate".
- sammanfattning: 2–3 meningar som står på egna ben: vem kunden är, vad Mindcamp gjorde och vad det gav. Ska gå att citera rakt av.
- om_kunden: 2–4 meningar om kundens verksamhet.
- utmaning: problemet före projektet, 1–3 stycken.
- losning: vad Mindcamp gjorde och med vilken teknik. Nämn teknikerna vid namn.
- resultat: konkreta effekter. Använd siffror när underlaget har dem.
- citat: kundcitat med namn och titel, om underlaget innehåller något.
- faq: 3–4 frågor som någon kan tänkas ställa till en AI-assistent om den här typen av projekt, med korta svar som utgår från caset.

Viktigt
- Hitta aldrig på siffror, citat, namn eller titlar. Där ett mätbart resultat eller ett kundcitat skulle stärka texten men saknas i underlaget, skriv en markering i hakparentes med versaler, t.ex. [SIFFRA: tidsbesparing per månad] eller [KUNDCITAT]. Markeringarna ersätts senare med kundens svar.
- Skriv påståenden som är tydliga utan sammanhang: "Mindcamp hjälpte Acme AB att ..." hellre än "Vi hjälpte dem att ...".
- Separera stycken med en tom rad. Ingen markdown.
```

`app/prompts/fragor.txt`:
```
Du hjälper Mindcamp att komplettera ett kundcase med information från kunden.

Du får konsultens underlag och ett utkast. Utkastet kan innehålla markeringar i hakparentes, t.ex. [SIFFRA: ...] eller [KUNDCITAT], där information saknas.

Skriv 3–5 frågor till kundens kontaktperson som fyller luckorna. Prioritera:
1. Mätbara resultat (tid, pengar, antal, kvalitet). Be om ungefärliga siffror om exakta saknas.
2. Ett citat: be kontaktpersonen beskriva med egna ord vad projektet har betytt, t.ex. "Vad skulle du säga till en kollega som funderar på något liknande?"
3. Övriga markeringar i utkastet.

Frågorna ska vara korta, på svenska, ställda till "du" eller "ni", och gå att besvara på några minuter. Fråga inte efter namn och titel – det fylls i separat.
```

`app/prompts/invavning.txt`:
```
Du uppdaterar ett kundcase för Mindcamp med kundens svar.

Du får utkastet, frågorna som ställdes och kundens svar: svar per fråga, kommentarer per sektion samt kontaktpersonens namn och titel.

Gör så här
- Ersätt markeringar i hakparentes med uppgifter från kundens svar. Markeringar som kunden inte gett underlag för tar du bort, och formulerar om texten utan dem.
- Använd kundens egna formuleringar som citat i fältet citat, med namn, titel och företag, t.ex.: "Citatet." – Förnamn Efternamn, Titel, Företag. Du får korta och språkgranska citat lätt men inte ändra innebörden.
- Följ kundens kommentarer per sektion när de rättar sakfel eller formuleringar.
- Lägg in mätbara resultat i resultat och, om de är centrala, även i sammanfattning.
- Uppdatera faq om svaren ger bättre underlag.
- Behåll i övrigt struktur, ton och längd. Hitta inte på något som inte finns i utkastet eller svaren.
- Separera stycken med en tom rad. Ingen markdown.
```

- [ ] **Steg 4: Skriv AI-modulen**

`app/ai.py`:
```python
import json
from pathlib import Path

import anthropic

from app import config
from app.sektioner import SEKTIONER

PROMPTMAPP = Path(__file__).parent / "prompts"

UTKAST_SCHEMA = {
    "type": "object",
    "properties": {
        "titel": {"type": "string"},
        **{nyckel: {"type": "string"} for nyckel, _ in SEKTIONER},
        "faq": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {"fraga": {"type": "string"}, "svar": {"type": "string"}},
                "required": ["fraga", "svar"],
                "additionalProperties": False,
            },
        },
    },
    "required": ["titel", *[nyckel for nyckel, _ in SEKTIONER], "faq"],
    "additionalProperties": False,
}

FRAGOR_SCHEMA = {
    "type": "object",
    "properties": {"fragor": {"type": "array", "items": {"type": "string"}}},
    "required": ["fragor"],
    "additionalProperties": False,
}


class AiFel(Exception):
    pass


def _anropa(promptfil, indata, schema):
    klient = anthropic.Anthropic()
    try:
        svar = klient.beta.messages.create(
            model=config.CLAUDE_MODEL,
            max_tokens=16000,
            system=(PROMPTMAPP / promptfil).read_text(encoding="utf-8"),
            messages=[{"role": "user", "content": json.dumps(indata, ensure_ascii=False, indent=2)}],
            output_config={"format": {"type": "json_schema", "schema": schema}},
            betas=["server-side-fallback-2026-07-01"],
            fallbacks="default",
        )
    except anthropic.APIError as e:
        raise AiFel(f"Anropet till Claude misslyckades: {e}") from e

    if svar.stop_reason != "end_turn":
        raise AiFel(f"Claude avbröt svaret ({svar.stop_reason})")
    text = next(b.text for b in svar.content if b.type == "text")
    return json.loads(text)


def skriv_utkast(formular):
    return _anropa("utkast.txt", formular, UTKAST_SCHEMA)


def skapa_fragor(formular, utkast):
    indata = {"underlag": formular, "utkast": utkast}
    return _anropa("fragor.txt", indata, FRAGOR_SCHEMA)["fragor"]


def vav_in_svar(utkast, fragor, kundsvar):
    indata = {"utkast": utkast, "fragor": fragor, "kundsvar": kundsvar}
    return _anropa("invavning.txt", indata, UTKAST_SCHEMA)
```

`fallbacks="default"` med betan `server-side-fallback-2026-07-01` gör att API:t automatiskt kör om anropet på en annan modell om Sonnet 5.5 nekar av säkerhetsskäl. Det ska inte hända för kundcase, men skyddar mot falsklarm.

- [ ] **Steg 5: Kör testerna**

Kör: `pytest tests/test_ai.py -v`
Förväntat: 5 passed

- [ ] **Steg 6: Commit**

```bash
git add app/ai.py app/prompts/ tests/test_ai.py
git commit -m "AI-modul med promptar för utkast, frågor och invävning"
```

---

### Task 5: Webbappen – lista och nytt case

**Filer:**
- Skapa: `app/mallar.py`, `app/main.py`, `app/routes_admin.py`, `app/routes_kund.py` (tom router), `app/templates/base.html`, `app/templates/inte_inloggad.html`, `app/templates/admin_lista.html`, `app/templates/admin_nytt.html`, `app/templates/admin_case.html`, `tests/hjalp.py`, `tests/test_admin.py`
- Ändra: `tests/conftest.py`

**Gränssnitt:**
- Konsumerar: `inloggad_konsult`, `Case`, `get_session`, `ai.skriv_utkast`, `ai.skapa_fragor`, `ai.AiFel`, `SEKTIONER`, `faq_till_text`, `hitta_markeringar`
- Producerar: `app.main.app`; `templates` med globals `SEKTIONER`, `STATUSTEXT`; routes `GET /admin`, `GET/POST /admin/nytt`, `GET /admin/case/{id}`, `POST /admin/case/{id}/generera`; hjälpfunktioner i `routes_admin`: `_hamta`, `_krav_status`, `_till_case`; fixtures `client`, `falsk_ai`; `tests.hjalp.UTKAST`, `tests.hjalp.FORMULAR`, `tests.hjalp.skapa_case()`

- [ ] **Steg 1: Lägg till testhjälp och fixtures**

`tests/hjalp.py`:
```python
from app.db import Case

UTKAST = {
    "titel": "En skalbar grund för automatisering",
    "sammanfattning": "Mindcamp hjälpte Acme AB att automatisera.",
    "om_kunden": "Acme AB säljer saker.",
    "utmaning": "Mycket manuellt arbete.",
    "losning": "Flöden i Power Automate.",
    "resultat": "Sparar [SIFFRA: timmar per månad].",
    "citat": "[KUNDCITAT]",
    "faq": [{"fraga": "Vad gjorde Mindcamp?", "svar": "Automatiserade flöden."}],
}

FORMULAR = {
    "kundnamn": "Acme AB",
    "kontakt_namn": "Anna Andersson",
    "kontakt_epost": "anna@acme.se",
    "utmaning": "Manuellt",
    "losning": "Automation",
    "teknik": "Power Automate",
    "resultat": "Snabbare",
    "anteckningar": "",
}


def skapa_case(session, **falt):
    standard = {
        "skapad_av": "test@mindcamp.se",
        "kundnamn": "Acme AB",
        "kontakt_namn": "Anna Andersson",
        "kontakt_epost": "anna@acme.se",
        "formular": {"kund": "Acme AB"},
        "utkast": dict(UTKAST),
        "fragor": ["Hur mycket tid sparar ni?", "Vad skulle du säga till en kollega?"],
    }
    standard.update(falt)
    case = Case(**standard)
    session.add(case)
    session.commit()
    return case
```

Lägg till i slutet av `tests/conftest.py`:
```python
from fastapi.testclient import TestClient

from app import ai, config
from app.main import app
from tests.hjalp import UTKAST


@pytest.fixture
def client(session_fabrik, monkeypatch):
    monkeypatch.setattr(config, "DEV_USER", "test@mindcamp.se")

    def test_session():
        with session_fabrik() as s:
            yield s

    app.dependency_overrides[db.get_session] = test_session
    yield TestClient(app)
    app.dependency_overrides.clear()


@pytest.fixture
def falsk_ai(monkeypatch):
    monkeypatch.setattr(ai, "skriv_utkast", lambda formular: dict(UTKAST))
    monkeypatch.setattr(ai, "skapa_fragor", lambda formular, utkast: ["Hur mycket tid sparar ni?", "Vad skulle du säga till en kollega?"])
```
(Flytta importerna upp till toppen av filen.) `TestClient(app)` utan `with` kör inte appens lifespan, så ingen riktig `kundcase.db` skapas under test.

- [ ] **Steg 2: Skriv de fallerande testerna**

`tests/test_admin.py`:
```python
from sqlalchemy import select

from app import ai, config
from app.db import Case
from tests.hjalp import FORMULAR, skapa_case


def test_admin_kraver_inloggning(client, monkeypatch):
    monkeypatch.setattr(config, "DEV_USER", "")
    svar = client.get("/admin")
    assert svar.status_code == 401
    assert "Logga in" in svar.text


def test_lista_visar_case(client, session):
    skapa_case(session, kundnamn="Kia Sverige")
    svar = client.get("/admin")
    assert svar.status_code == 200
    assert "Kia Sverige" in svar.text


def test_nytt_case_skapar_utkast_och_fragor(client, session, falsk_ai):
    svar = client.post("/admin/nytt", data=FORMULAR, follow_redirects=False)
    assert svar.status_code == 303

    case = session.scalars(select(Case)).one()
    assert case.status == "utkast"
    assert case.skapad_av == "test@mindcamp.se"
    assert case.formular["teknik"] == "Power Automate"
    assert case.utkast["titel"] == "En skalbar grund för automatisering"
    assert len(case.fragor) == 2


def test_casesidan_visar_utkast_och_markeringar(client, session):
    case = skapa_case(session)
    svar = client.get(f"/admin/case/{case.id}")
    assert "En skalbar grund för automatisering" in svar.text
    assert "[KUNDCITAT]" in svar.text
    assert "Hur mycket tid sparar ni?" in svar.text


def test_ai_fel_sparar_formular_och_visar_fel(client, session, monkeypatch):
    def fel(formular):
        raise ai.AiFel("Claude svarar inte")
    monkeypatch.setattr(ai, "skriv_utkast", fel)

    svar = client.post("/admin/nytt", data=FORMULAR)
    assert "Claude svarar inte" in svar.text
    assert "Skriv utkast med AI" in svar.text

    case = session.scalars(select(Case)).one()
    assert case.formular["utmaning"] == "Manuellt"
    assert case.utkast == {}


def test_generera_igen_efter_fel(client, session, falsk_ai):
    case = skapa_case(session, utkast={}, fragor=[])
    client.post(f"/admin/case/{case.id}/generera")
    session.refresh(case)
    assert case.utkast["titel"] == "En skalbar grund för automatisering"


def test_okant_case_ger_404(client):
    assert client.get("/admin/case/999").status_code == 404
```

- [ ] **Steg 3: Kör testerna och se att de fallerar**

Kör: `pytest tests/test_admin.py -v`
Förväntat: FAIL med `ModuleNotFoundError: No module named 'app.main'`

- [ ] **Steg 4: Skriv mallar och app**

`app/mallar.py`:
```python
from pathlib import Path

from fastapi.templating import Jinja2Templates

from app.sektioner import SEKTIONER

STATUSTEXT = {
    "utkast": "Utkast",
    "hos_kund": "Hos kund",
    "kund_svarat": "Kund har svarat",
    "klar": "Klar",
}

templates = Jinja2Templates(directory=Path(__file__).parent / "templates")
templates.env.globals["SEKTIONER"] = SEKTIONER
templates.env.globals["STATUSTEXT"] = STATUSTEXT
```

`app/routes_kund.py` (fylls i Task 7):
```python
from fastapi import APIRouter

router = APIRouter()
```

`app/main.py`:
```python
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.exception_handlers import http_exception_handler
from fastapi.responses import RedirectResponse
from starlette.exceptions import HTTPException

from app import db
from app.mallar import templates
from app.routes_admin import router as admin_router
from app.routes_kund import router as kund_router


@asynccontextmanager
async def lifespan(app):
    db.init_db()
    yield


app = FastAPI(lifespan=lifespan)
app.include_router(admin_router)
app.include_router(kund_router)


@app.exception_handler(HTTPException)
async def http_fel(request: Request, exc: HTTPException):
    if exc.status_code == 401:
        return templates.TemplateResponse(request, "inte_inloggad.html", status_code=401)
    return await http_exception_handler(request, exc)


@app.get("/")
def start():
    return RedirectResponse("/admin")
```

`app/routes_admin.py`:
```python
from urllib.parse import quote

from fastapi import APIRouter, Depends, Form, HTTPException, Request
from fastapi.responses import RedirectResponse
from sqlalchemy import select
from sqlalchemy.orm import Session

from app import ai, config
from app.auth import inloggad_konsult
from app.db import Case, get_session
from app.mallar import templates
from app.sektioner import faq_till_text, hitta_markeringar

router = APIRouter(prefix="/admin", dependencies=[Depends(inloggad_konsult)])


def _hamta(session, case_id):
    case = session.get(Case, case_id)
    if case is None:
        raise HTTPException(status_code=404, detail="Caset finns inte")
    return case


def _krav_status(case, status):
    if case.status != status:
        raise HTTPException(status_code=409, detail=f"Caset har status {case.status}")


def _till_case(case_id, fel=None):
    url = f"/admin/case/{case_id}"
    if fel:
        url += "?fel=" + quote(fel)
    return RedirectResponse(url, status_code=303)


def _generera(session, case):
    try:
        utkast = ai.skriv_utkast(case.formular)
        fragor = ai.skapa_fragor(case.formular, utkast)
    except ai.AiFel as e:
        return _till_case(case.id, str(e))
    case.utkast = utkast
    case.fragor = fragor
    session.commit()
    return _till_case(case.id)


@router.get("")
def lista(
    request: Request,
    session: Session = Depends(get_session),
    anvandare: str = Depends(inloggad_konsult),
):
    cases = session.scalars(select(Case).order_by(Case.uppdaterad.desc())).all()
    return templates.TemplateResponse(request, "admin_lista.html", {"cases": cases, "anvandare": anvandare})


@router.get("/nytt")
def nytt_formular(request: Request):
    return templates.TemplateResponse(request, "admin_nytt.html")


@router.post("/nytt")
def skapa(
    kundnamn: str = Form(...),
    kontakt_namn: str = Form(""),
    kontakt_epost: str = Form(""),
    utmaning: str = Form(""),
    losning: str = Form(""),
    teknik: str = Form(""),
    resultat: str = Form(""),
    anteckningar: str = Form(""),
    session: Session = Depends(get_session),
    anvandare: str = Depends(inloggad_konsult),
):
    case = Case(
        skapad_av=anvandare,
        kundnamn=kundnamn,
        kontakt_namn=kontakt_namn,
        kontakt_epost=kontakt_epost,
        formular={
            "kund": kundnamn,
            "utmaning": utmaning,
            "losning": losning,
            "teknik": teknik,
            "resultat": resultat,
            "anteckningar": anteckningar,
        },
    )
    session.add(case)
    session.commit()
    return _generera(session, case)


@router.post("/case/{case_id}/generera")
def generera(case_id: int, session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    _krav_status(case, "utkast")
    return _generera(session, case)


@router.get("/case/{case_id}")
def visa(case_id: int, request: Request, fel: str = "", session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    return templates.TemplateResponse(request, "admin_case.html", {
        "case": case,
        "fel": fel,
        "faq_text": faq_till_text(case.utkast.get("faq", [])),
        "markeringar": hitta_markeringar(case.utkast),
        "kundlank": f"{config.BASE_URL}/c/{case.token}" if case.token else "",
    })
```

- [ ] **Steg 5: Skriv HTML-mallarna**

`app/templates/base.html`:
```html
<!doctype html>
<html lang="sv">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{% block titel %}Kundcase{% endblock %} – Mindcamp</title>
<style>
  body { font-family: system-ui, sans-serif; max-width: 860px; margin: 0 auto; padding: 24px 16px; color: #1a1a1a; line-height: 1.5; }
  h1 { font-size: 1.6rem; }
  label { display: block; font-weight: 600; margin-top: 16px; }
  input[type=text], input[type=email], textarea { width: 100%; box-sizing: border-box; padding: 8px; font: inherit; border: 1px solid #bbb; border-radius: 4px; }
  textarea { min-height: 120px; }
  button { margin-top: 16px; margin-right: 8px; padding: 8px 16px; font: inherit; cursor: pointer; }
  table { width: 100%; border-collapse: collapse; }
  td, th { text-align: left; padding: 6px 8px; border-bottom: 1px solid #ddd; }
  .fel { background: #fde8e8; border: 1px solid #e0a0a0; padding: 12px; border-radius: 4px; margin: 16px 0; }
  .info { background: #eef4fb; border: 1px solid #b6cde8; padding: 12px; border-radius: 4px; margin: 16px 0; }
  .status { font-size: .85rem; font-weight: normal; background: #eee; padding: 2px 8px; border-radius: 10px; }
  pre { white-space: pre-wrap; font: inherit; background: #f6f6f6; padding: 12px; border-radius: 4px; }
</style>
</head>
<body>
{% block innehall %}{% endblock %}
</body>
</html>
```

`app/templates/inte_inloggad.html`:
```html
{% extends "base.html" %}
{% block titel %}Inte inloggad{% endblock %}
{% block innehall %}
<h1>Inte inloggad</h1>
<p><a href="/.auth/login/aad?post_login_redirect_uri=/admin">Logga in med ditt Mindcamp-konto</a></p>
{% endblock %}
```

`app/templates/admin_lista.html`:
```html
{% extends "base.html" %}
{% block titel %}Kundcase{% endblock %}
{% block innehall %}
<h1>Kundcase</h1>
<p>Inloggad som {{ anvandare }}</p>
<p><a href="/admin/nytt">+ Nytt case</a></p>
<table>
  <tr><th>Kund</th><th>Status</th><th>Skapad av</th><th>Uppdaterad</th></tr>
  {% for c in cases %}
  <tr>
    <td><a href="/admin/case/{{ c.id }}">{{ c.kundnamn }}</a></td>
    <td><span class="status">{{ STATUSTEXT[c.status] }}</span></td>
    <td>{{ c.skapad_av }}</td>
    <td>{{ c.uppdaterad.strftime("%Y-%m-%d %H:%M") }}</td>
  </tr>
  {% else %}
  <tr><td colspan="4">Inga case ännu.</td></tr>
  {% endfor %}
</table>
{% endblock %}
```

`app/templates/admin_nytt.html`:
```html
{% extends "base.html" %}
{% block titel %}Nytt case{% endblock %}
{% block innehall %}
<p><a href="/admin">← Alla case</a></p>
<h1>Nytt kundcase</h1>
<form method="post">
  <label>Kund</label>
  <input type="text" name="kundnamn" required>
  <label>Kontaktperson hos kunden</label>
  <input type="text" name="kontakt_namn">
  <label>Kontaktpersonens e-post</label>
  <input type="email" name="kontakt_epost">
  <label>Utmaning – vad var problemet?</label>
  <textarea name="utmaning"></textarea>
  <label>Lösning – vad gjorde vi?</label>
  <textarea name="losning"></textarea>
  <label>Teknik</label>
  <input type="text" name="teknik" placeholder="t.ex. Power Automate, Power Apps">
  <label>Resultat – vad gav det? Siffror om du har dem.</label>
  <textarea name="resultat"></textarea>
  <label>Övrigt underlag – klistra in projektbeskrivning, mejl, anteckningar</label>
  <textarea name="anteckningar" style="min-height:200px"></textarea>
  <button>Skriv utkast med AI</button>
  <p>Det tar ungefär en minut.</p>
</form>
{% endblock %}
```

`app/templates/admin_case.html`:
```html
{% extends "base.html" %}
{% block titel %}{{ case.kundnamn }}{% endblock %}
{% block innehall %}
<p><a href="/admin">← Alla case</a></p>
<h1>{{ case.kundnamn }} <span class="status">{{ STATUSTEXT[case.status] }}</span></h1>

{% if fel %}
<div class="fel">{{ fel }}</div>
{% endif %}

{% if not case.utkast %}
<div class="info">
  Inget utkast ännu.
  <form method="post" action="/admin/case/{{ case.id }}/generera"><button>Skriv utkast med AI</button></form>
</div>
{% endif %}

{% if case.status == "hos_kund" %}
<div class="info">
  Skicka den här länken till {{ case.kontakt_namn }} ({{ case.kontakt_epost }}):<br>
  <strong>{{ kundlank }}</strong>
</div>
{% endif %}

{% if case.status == "kund_svarat" %}
<h2>Kundens svar</h2>
<p><strong>{{ case.kundsvar.namn }}</strong>, {{ case.kundsvar.titel }}</p>
{% for s in case.kundsvar.svar %}
<p><strong>{{ s.fraga }}</strong><br>{{ s.svar or "–" }}</p>
{% endfor %}
{% for nyckel, rubrik in SEKTIONER %}{% if case.kundsvar.kommentarer[nyckel] %}
<p><strong>Kommentar på {{ rubrik }}:</strong> {{ case.kundsvar.kommentarer[nyckel] }}</p>
{% endif %}{% endfor %}
<form method="post" action="/admin/case/{{ case.id }}/vav-in"><button>Väv in kundens svar med AI</button></form>
{% endif %}

{% if markeringar and case.status != "hos_kund" %}
<div class="fel">Texten innehåller markeringar som behöver åtgärdas: {{ markeringar | join(", ") }}</div>
{% endif %}

{% if case.utkast %}
  {% if case.status in ("utkast", "kund_svarat") %}
  <form method="post" action="/admin/case/{{ case.id }}/spara">
    <label>Titel</label>
    <input type="text" name="titel" value="{{ case.utkast.titel }}">
    {% for nyckel, rubrik in SEKTIONER %}
    <label>{{ rubrik }}</label>
    <textarea name="{{ nyckel }}">{{ case.utkast[nyckel] }}</textarea>
    {% endfor %}
    <label>FAQ (F: fråga / S: svar)</label>
    <textarea name="faq">{{ faq_text }}</textarea>
    {% if case.status == "utkast" %}
    <label>Frågor till kunden (en per rad)</label>
    <textarea name="fragor">{{ case.fragor | join("\n") }}</textarea>
    {% endif %}
    <button>Spara</button>
    {% if case.status == "utkast" %}
    <button formaction="/admin/case/{{ case.id }}/skicka">Spara och skicka till kund</button>
    {% else %}
    <button formaction="/admin/case/{{ case.id }}/klar">Spara och markera som klar</button>
    {% endif %}
  </form>
  {% else %}
  <h2>{{ case.utkast.titel }}</h2>
  {% for nyckel, rubrik in SEKTIONER %}
  <h3>{{ rubrik }}</h3>
  <pre>{{ case.utkast[nyckel] }}</pre>
  {% endfor %}
  {% if case.status == "klar" %}
  <p><a href="/admin/case/{{ case.id }}/export">Visa export</a></p>
  {% endif %}
  {% endif %}
{% endif %}
{% endblock %}
```
Routerna `spara`, `skicka`, `vav-in`, `klar` och `export` som mallen pekar på läggs till i Task 6–8.

- [ ] **Steg 6: Kör testerna**

Kör: `pytest -v`
Förväntat: alla tester passerar (7 nya i `test_admin.py`)

- [ ] **Steg 7: Commit**

```bash
git add app/ tests/
git commit -m "Webbapp: lista case och skapa nytt case med AI-utkast"
```

---

### Task 6: Redigera utkast och skicka till kund

**Filer:**
- Ändra: `app/routes_admin.py`
- Skapa: `tests/test_admin_skicka.py`
- Ändra: `tests/hjalp.py`

**Gränssnitt:**
- Konsumerar: `_hamta`, `_krav_status`, `_till_case`, `SEKTIONER`, `text_till_faq`
- Producerar: `async _spara(request, case)`; routes `POST /admin/case/{id}/spara`, `POST /admin/case/{id}/skicka`; `tests.hjalp.spara_data(**over) -> dict`

- [ ] **Steg 1: Lägg till testhjälp**

Lägg till i `tests/hjalp.py`:
```python
def spara_data(**over):
    data = {
        "titel": "Ny titel",
        "sammanfattning": "S",
        "om_kunden": "O",
        "utmaning": "U",
        "losning": "L",
        "resultat": "R",
        "citat": "C",
        "faq": "F: Fråga?\nS: Svar.",
        "fragor": "Fråga 1\n\nFråga 2\nFråga 3",
    }
    data.update(over)
    return data
```

- [ ] **Steg 2: Skriv de fallerande testerna**

`tests/test_admin_skicka.py`:
```python
from app import config
from tests.hjalp import skapa_case, spara_data


def test_spara_uppdaterar_utkast_och_fragor(client, session):
    case = skapa_case(session)
    svar = client.post(f"/admin/case/{case.id}/spara", data=spara_data(), follow_redirects=False)
    assert svar.status_code == 303

    session.refresh(case)
    assert case.utkast["titel"] == "Ny titel"
    assert case.utkast["faq"] == [{"fraga": "Fråga?", "svar": "Svar."}]
    assert case.fragor == ["Fråga 1", "Fråga 2", "Fråga 3"]
    assert case.status == "utkast"


def test_skicka_sparar_andringar_och_skapar_kundlank(client, session):
    case = skapa_case(session)
    client.post(f"/admin/case/{case.id}/skicka", data=spara_data(titel="Ändrad före skick"))

    session.refresh(case)
    assert case.status == "hos_kund"
    assert case.utkast["titel"] == "Ändrad före skick"
    assert len(case.token) >= 40
    assert case.token_aktiv is True

    sida = client.get(f"/admin/case/{case.id}")
    assert f"{config.BASE_URL}/c/{case.token}" in sida.text


def test_skicka_tva_ganger_ger_409(client, session):
    case = skapa_case(session)
    client.post(f"/admin/case/{case.id}/skicka", data=spara_data())
    svar = client.post(f"/admin/case/{case.id}/skicka", data=spara_data())
    assert svar.status_code == 409


def test_skicka_utan_utkast_ger_409(client, session):
    case = skapa_case(session, utkast={})
    assert client.post(f"/admin/case/{case.id}/skicka", data=spara_data()).status_code == 409


def test_spara_nar_hos_kund_ger_409(client, session):
    case = skapa_case(session, status="hos_kund", token="t" * 43, token_aktiv=True)
    assert client.post(f"/admin/case/{case.id}/spara", data=spara_data()).status_code == 409
```

- [ ] **Steg 3: Kör testerna och se att de fallerar**

Kör: `pytest tests/test_admin_skicka.py -v`
Förväntat: FAIL med 404/405 för `/spara` och `/skicka`

- [ ] **Steg 4: Lägg till routerna**

I `app/routes_admin.py`, lägg till `import secrets` överst och utöka importen från `app.sektioner`:
```python
import secrets
...
from app.sektioner import SEKTIONER, faq_till_text, hitta_markeringar, text_till_faq
```

Lägg till i slutet av filen:
```python
async def _spara(request, case):
    form = await request.form()
    utkast = {"titel": form.get("titel", "")}
    for nyckel, _ in SEKTIONER:
        utkast[nyckel] = form.get(nyckel, "")
    utkast["faq"] = text_till_faq(form.get("faq", ""))
    case.utkast = utkast
    if "fragor" in form:
        case.fragor = [rad.strip() for rad in form["fragor"].splitlines() if rad.strip()]


@router.post("/case/{case_id}/spara")
async def spara(case_id: int, request: Request, session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    if case.status not in ("utkast", "kund_svarat"):
        raise HTTPException(status_code=409, detail=f"Caset har status {case.status}")
    await _spara(request, case)
    session.commit()
    return _till_case(case.id)


@router.post("/case/{case_id}/skicka")
async def skicka(case_id: int, request: Request, session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    _krav_status(case, "utkast")
    if not case.utkast:
        raise HTTPException(status_code=409, detail="Caset saknar utkast")
    await _spara(request, case)
    case.token = secrets.token_urlsafe(32)
    case.token_aktiv = True
    case.status = "hos_kund"
    session.commit()
    return _till_case(case.id)
```

- [ ] **Steg 5: Kör testerna**

Kör: `pytest -v`
Förväntat: alla passerar

- [ ] **Steg 6: Commit**

```bash
git add app/routes_admin.py tests/
git commit -m "Redigera utkast och skicka kundlänk"
```

---

### Task 7: Kundsidan

**Filer:**
- Ändra: `app/routes_kund.py`
- Skapa: `app/templates/kund.html`, `app/templates/kund_tack.html`, `app/templates/inaktiv.html`, `tests/test_kund.py`

**Gränssnitt:**
- Konsumerar: `Case`, `get_session`, `nu`, `SEKTIONER`, `templates`
- Producerar: `GET /c/{token}`, `POST /c/{token}`; `case.kundsvar` på formen `{"namn": str, "titel": str, "svar": [{"fraga": str, "svar": str}], "kommentarer": {sektionsnyckel: str}}`

- [ ] **Steg 1: Skriv de fallerande testerna**

`tests/test_kund.py`:
```python
from tests.hjalp import skapa_case

TOKEN = "k" * 43


def _hos_kund(session, **falt):
    return skapa_case(session, status="hos_kund", token=TOKEN, token_aktiv=True, **falt)


SVAR = {
    "namn": "Anna Andersson",
    "titel": "CIO",
    "svar_0": "Ungefär 20 timmar i månaden",
    "svar_1": "",
    "kommentar_utmaning": "Det var tre team, inte två.",
}


def test_kundsidan_visar_case_fragor_och_samtycke(client, session):
    _hos_kund(session)
    svar = client.get(f"/c/{TOKEN}")
    assert svar.status_code == 200
    assert "En skalbar grund för automatisering" in svar.text
    assert "Hur mycket tid sparar ni?" in svar.text
    assert "kan komma att citeras" in svar.text


def test_okand_token_ger_404(client):
    svar = client.get("/c/finns-inte")
    assert svar.status_code == 404
    assert "inte längre aktiv" in svar.text


def test_inaktiv_token_ger_404(client, session):
    _hos_kund(session).token_aktiv = False
    session.commit()
    assert client.get(f"/c/{TOKEN}").status_code == 404


def test_kund_skickar_svar(client, session):
    case = _hos_kund(session)
    svar = client.post(f"/c/{TOKEN}", data=SVAR)
    assert svar.status_code == 200
    assert "Tack" in svar.text

    session.refresh(case)
    assert case.status == "kund_svarat"
    assert case.kund_svarade is not None
    assert case.kundsvar["namn"] == "Anna Andersson"
    assert case.kundsvar["svar"][0] == {"fraga": "Hur mycket tid sparar ni?", "svar": "Ungefär 20 timmar i månaden"}
    assert case.kundsvar["svar"][1]["svar"] == ""
    assert case.kundsvar["kommentarer"]["utmaning"] == "Det var tre team, inte två."
    assert case.kundsvar["kommentarer"]["losning"] == ""


def test_andra_inskick_avvisas(client, session):
    case = _hos_kund(session)
    client.post(f"/c/{TOKEN}", data=SVAR)
    svar = client.post(f"/c/{TOKEN}", data={**SVAR, "svar_0": "Ändrat"})
    assert svar.status_code == 409

    session.refresh(case)
    assert case.kundsvar["svar"][0]["svar"] == "Ungefär 20 timmar i månaden"


def test_kundsidan_ar_skrivskyddad_efter_svar(client, session):
    _hos_kund(session)
    client.post(f"/c/{TOKEN}", data=SVAR)
    svar = client.get(f"/c/{TOKEN}")
    assert "Tack" in svar.text
    assert "<textarea" not in svar.text


def test_html_i_kundsvar_escapas_hos_konsulten(client, session):
    case = _hos_kund(session)
    client.post(f"/c/{TOKEN}", data={**SVAR, "svar_0": "<script>alert(1)</script>"})
    sida = client.get(f"/admin/case/{case.id}")
    assert "<script>alert(1)</script>" not in sida.text
    assert "&lt;script&gt;" in sida.text
```

- [ ] **Steg 2: Kör testerna och se att de fallerar**

Kör: `pytest tests/test_kund.py -v`
Förväntat: FAIL med 404 utan "inte längre aktiv" / 405 på POST

- [ ] **Steg 3: Skriv kundroutern**

`app/routes_kund.py`:
```python
from fastapi import APIRouter, Depends, Request
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.db import Case, get_session, nu
from app.mallar import templates
from app.sektioner import SEKTIONER

router = APIRouter()


def _hamta_aktiv(session, token):
    case = session.scalar(select(Case).where(Case.token == token))
    if case is None or not case.token_aktiv:
        return None
    return case


@router.get("/c/{token}")
def visa(token: str, request: Request, session: Session = Depends(get_session)):
    case = _hamta_aktiv(session, token)
    if case is None:
        return templates.TemplateResponse(request, "inaktiv.html", status_code=404)
    if case.status != "hos_kund":
        return templates.TemplateResponse(request, "kund_tack.html", {"case": case})
    return templates.TemplateResponse(request, "kund.html", {"case": case})


@router.post("/c/{token}")
async def svara(token: str, request: Request, session: Session = Depends(get_session)):
    case = _hamta_aktiv(session, token)
    if case is None:
        return templates.TemplateResponse(request, "inaktiv.html", status_code=404)
    if case.status != "hos_kund":
        return templates.TemplateResponse(request, "kund_tack.html", {"case": case}, status_code=409)

    form = await request.form()
    case.kundsvar = {
        "namn": form.get("namn", "").strip(),
        "titel": form.get("titel", "").strip(),
        "svar": [
            {"fraga": fraga, "svar": form.get(f"svar_{i}", "").strip()}
            for i, fraga in enumerate(case.fragor)
        ],
        "kommentarer": {nyckel: form.get(f"kommentar_{nyckel}", "").strip() for nyckel, _ in SEKTIONER},
    }
    case.status = "kund_svarat"
    case.kund_svarade = nu()
    session.commit()
    return templates.TemplateResponse(request, "kund_tack.html", {"case": case})
```

- [ ] **Steg 4: Skriv kundmallarna**

`app/templates/kund.html`:
```html
{% extends "base.html" %}
{% block titel %}Granska kundcase{% endblock %}
{% block innehall %}
<h1>{{ case.utkast.titel }}</h1>
<div class="info">
  Mindcamp har skrivit ett utkast till kundcase om ert samarbete. Läs igenom texten, lämna gärna kommentarer
  och svara på frågorna längst ned. Det tar ungefär tio minuter.<br><br>
  <strong>Dina svar kan komma att citeras i det publicerade caset med namn och titel.</strong>
</div>
<form method="post">
  {% for nyckel, rubrik in SEKTIONER %}
  <h2>{{ rubrik }}</h2>
  <pre>{{ case.utkast[nyckel] }}</pre>
  <label>Kommentar (valfritt)</label>
  <textarea name="kommentar_{{ nyckel }}" style="min-height:60px"></textarea>
  {% endfor %}

  <h2>Frågor till dig</h2>
  {% for fraga in case.fragor %}
  <label>{{ fraga }}</label>
  <textarea name="svar_{{ loop.index0 }}"></textarea>
  {% endfor %}

  <h2>Om dig</h2>
  <label>Namn</label>
  <input type="text" name="namn" value="{{ case.kontakt_namn }}" required>
  <label>Titel</label>
  <input type="text" name="titel" required>

  <button>Skicka in</button>
</form>
{% endblock %}
```

`app/templates/kund_tack.html`:
```html
{% extends "base.html" %}
{% block titel %}Tack{% endblock %}
{% block innehall %}
<h1>Tack!</h1>
<p>Dina svar är inskickade. Mindcamp hör av sig om något behöver kompletteras.</p>
{% if case.kundsvar %}
<h2>Det du skickade in</h2>
{% for s in case.kundsvar.svar %}
<p><strong>{{ s.fraga }}</strong><br>{{ s.svar or "–" }}</p>
{% endfor %}
{% endif %}
{% endblock %}
```

`app/templates/inaktiv.html`:
```html
{% extends "base.html" %}
{% block titel %}Länken är inte aktiv{% endblock %}
{% block innehall %}
<h1>Länken är inte längre aktiv</h1>
<p>Hör av dig till din kontaktperson på Mindcamp om du behöver komma åt caset.</p>
{% endblock %}
```

- [ ] **Steg 5: Kör testerna**

Kör: `pytest -v`
Förväntat: alla passerar

- [ ] **Steg 6: Commit**

```bash
git add app/ tests/test_kund.py
git commit -m "Kundsida med kommentarer, frågor och engångsinskick"
```

---

### Task 8: Slutredigering, klar och export

**Filer:**
- Ändra: `app/routes_admin.py`
- Skapa: `app/templates/admin_export.html`, `tests/test_admin_slut.py`

**Gränssnitt:**
- Konsumerar: `ai.vav_in_svar`, `ai.AiFel`, `_spara`, `_hamta`, `_krav_status`, `_till_case`, `bygg_export`
- Producerar: `POST /admin/case/{id}/vav-in`, `POST /admin/case/{id}/klar`, `GET /admin/case/{id}/export`

- [ ] **Steg 1: Skriv de fallerande testerna**

`tests/test_admin_slut.py`:
```python
from app import ai
from tests.hjalp import UTKAST, skapa_case, spara_data

KUNDSVAR = {
    "namn": "Anna Andersson",
    "titel": "CIO",
    "svar": [{"fraga": "Hur mycket tid sparar ni?", "svar": "20 timmar"}],
    "kommentarer": {},
}


def _kund_svarat(session, **falt):
    return skapa_case(session, status="kund_svarat", token="s" * 43, token_aktiv=True, kundsvar=KUNDSVAR, **falt)


def test_vav_in_uppdaterar_utkast(client, session, monkeypatch):
    mottaget = {}

    def vav_in(utkast, fragor, kundsvar):
        mottaget["kundsvar"] = kundsvar
        return {**utkast, "titel": "Invävd", "resultat": "Sparar 20 timmar per månad."}

    monkeypatch.setattr(ai, "vav_in_svar", vav_in)
    case = _kund_svarat(session)
    client.post(f"/admin/case/{case.id}/vav-in")

    session.refresh(case)
    assert case.utkast["titel"] == "Invävd"
    assert case.status == "kund_svarat"
    assert mottaget["kundsvar"]["namn"] == "Anna Andersson"


def test_vav_in_fel_behaller_utkast(client, session, monkeypatch):
    def fel(utkast, fragor, kundsvar):
        raise ai.AiFel("Timeout")

    monkeypatch.setattr(ai, "vav_in_svar", fel)
    case = _kund_svarat(session)
    svar = client.post(f"/admin/case/{case.id}/vav-in")

    assert "Timeout" in svar.text
    session.refresh(case)
    assert case.utkast == UTKAST


def test_vav_in_fore_kundsvar_ger_409(client, session):
    case = skapa_case(session)
    assert client.post(f"/admin/case/{case.id}/vav-in").status_code == 409


def test_klar_sparar_stanger_lank_och_visar_export(client, session):
    case = _kund_svarat(session)
    svar = client.post(f"/admin/case/{case.id}/klar", data=spara_data(titel="Slutlig titel"))

    assert svar.status_code == 200
    assert "Slutlig titel" in svar.text
    assert "application/ld+json" in svar.text

    session.refresh(case)
    assert case.status == "klar"
    assert case.utkast["titel"] == "Slutlig titel"
    assert case.token_aktiv is False
    assert client.get(f"/c/{case.token}").status_code == 404


def test_export_varnar_for_markeringar(client, session):
    case = skapa_case(session, status="klar")
    svar = client.get(f"/admin/case/{case.id}/export")
    assert "[KUNDCITAT]" in svar.text
    assert "markeringar" in svar.text


def test_export_fore_klar_ger_409(client, session):
    case = skapa_case(session)
    assert client.get(f"/admin/case/{case.id}/export").status_code == 409
```

- [ ] **Steg 2: Kör testerna och se att de fallerar**

Kör: `pytest tests/test_admin_slut.py -v`
Förväntat: FAIL med 404/405 för `/vav-in`, `/klar`, `/export`

- [ ] **Steg 3: Lägg till routerna**

I `app/routes_admin.py`, lägg till importen:
```python
from app.export import bygg_export
```

Lägg till i slutet av filen:
```python
@router.post("/case/{case_id}/vav-in")
def vav_in(case_id: int, session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    _krav_status(case, "kund_svarat")
    try:
        case.utkast = ai.vav_in_svar(case.utkast, case.fragor, case.kundsvar)
    except ai.AiFel as e:
        return _till_case(case.id, str(e))
    session.commit()
    return _till_case(case.id)


@router.post("/case/{case_id}/klar")
async def klar(case_id: int, request: Request, session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    _krav_status(case, "kund_svarat")
    await _spara(request, case)
    case.status = "klar"
    case.token_aktiv = False
    session.commit()
    return RedirectResponse(f"/admin/case/{case.id}/export", status_code=303)


@router.get("/case/{case_id}/export")
def export(case_id: int, request: Request, session: Session = Depends(get_session)):
    case = _hamta(session, case_id)
    _krav_status(case, "klar")
    return templates.TemplateResponse(request, "admin_export.html", {"case": case, "export": bygg_export(case)})
```

- [ ] **Steg 4: Skriv exportmallen**

`app/templates/admin_export.html`:
```html
{% extends "base.html" %}
{% block titel %}Export – {{ case.kundnamn }}{% endblock %}
{% block innehall %}
<p><a href="/admin/case/{{ case.id }}">← Tillbaka</a></p>
<h1>Export: {{ export.titel }}</h1>

{% if export.markeringar %}
<div class="fel">Texten innehåller fortfarande markeringar som behöver åtgärdas innan publicering: {{ export.markeringar | join(", ") }}</div>
{% endif %}

<h2>Text per sektion</h2>
<h3>Titel</h3>
<pre>{{ export.titel }}</pre>
{% for rubrik, text in export.sektioner %}
<h3>{{ rubrik }}</h3>
<pre>{{ text }}</pre>
{% endfor %}

<h3>FAQ</h3>
{% for p in export.faq %}
<pre><strong>{{ p.fraga }}</strong>
{{ p.svar }}</pre>
{% endfor %}

<h2>SEO</h2>
<h3>Metatitel</h3>
<pre>{{ export.meta_titel }}</pre>
<h3>Metabeskrivning</h3>
<pre>{{ export.meta_beskrivning }}</pre>
<h3>Strukturerad data</h3>
<p>Klistra in i sidans head eller i en HTML-ruta:</p>
<pre>&lt;script type="application/ld+json"&gt;
{{ export.json_ld }}
&lt;/script&gt;</pre>
{% endblock %}
```

- [ ] **Steg 5: Kör testerna**

Kör: `pytest -v`
Förväntat: alla passerar

- [ ] **Steg 6: Commit**

```bash
git add app/ tests/test_admin_slut.py
git commit -m "Väv in kundsvar, markera klar och exportera"
```

---

### Task 9: README och manuell genomkörning

**Filer:**
- Skapa: `README.md`

- [ ] **Steg 1: Skriv README**

`README.md`:
````markdown
# Kundcase-generator

Intern app för Mindcamp: konsulten får ett AI-skrivet kundcase, kunden kompletterar via en länk och konsulten exporterar text och SEO-paket till hemsidan.

## Kom igång lokalt

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fyll i ANTHROPIC_API_KEY
uvicorn app.main:app --reload
```

Öppna http://localhost:8000/admin. Lokalt är du inloggad som `DEV_USER`.

## Test

```bash
pytest
```

## Flöde

1. Nytt case → AI skriver utkast och frågor till kunden
2. Redigera → "Spara och skicka till kund" → mejla länken själv
3. Kunden kommenterar och svarar
4. "Väv in kundens svar med AI" → finputsa → "Spara och markera som klar"
5. Exportsidan: kopiera text, meta och JSON-LD till hemsidan

Promptarna ligger i `app/prompts/` och kan justeras utan kodändringar.
````

- [ ] **Steg 2: Kör hela flödet manuellt**

```bash
source venv/bin/activate
uvicorn app.main:app --reload
```
1. Skapa ett case med riktigt underlag (gärna Jakobs case).
2. Kontrollera att utkastet följer mallen, att markeringar finns där siffror/citat saknas och att frågorna är rimliga.
3. Klicka "Spara och skicka till kund", öppna kundlänken i ett privat fönster, svara och skicka in.
4. Öppna länken igen – sidan ska vara skrivskyddad.
5. Väv in, markera klar, kontrollera exporten. Klistra in JSON-LD i https://validator.schema.org och kontrollera att den validerar.
6. Öppna kundlänken – ska ge "Länken är inte längre aktiv".

Om Claude-anropet ger 400 på `fallbacks`/`betas`: ta bort de två raderna i `app/ai.py` och anropa `klient.messages.create(...)` i stället för `klient.beta.messages.create(...)`.

- [ ] **Steg 3: Commit**

```bash
git add README.md
git commit -m "README med kom igång och flöde"
git push
```

---

### Task 10: Driftsättning på Azure

Görs tillsammans med Jakob – kräver `az login` och beslut om resursnamn. Inga kodändringar utöver `startup.txt`.

**Filer:**
- Skapa: `startup.txt`

- [ ] **Steg 1: Startkommando**

`startup.txt`:
```
gunicorn -w 2 -k uvicorn.workers.UvicornWorker --timeout 300 app.main:app
```
`--timeout 300` behövs eftersom ett utkast + frågor kan ta över en minut; gunicorns standard (30 s) skulle döda anropet.

- [ ] **Steg 2: Skapa resurser**

```bash
az login
az group create -n rg-kundcase -l swedencentral

az postgres flexible-server create -g rg-kundcase -n pg-kundcase-mindcamp -l swedencentral \
  --tier Burstable --sku-name Standard_B1ms --storage-size 32 --version 16 \
  --admin-user kundcaseadmin --admin-password '<starkt lösenord>' --public-access 0.0.0.0
az postgres flexible-server db create -g rg-kundcase -s pg-kundcase-mindcamp -d kundcase
```
`--public-access 0.0.0.0` släpper bara in Azure-tjänster.

- [ ] **Steg 3: Driftsätt appen**

```bash
az webapp up -g rg-kundcase -n kundcase-mindcamp -l swedencentral --runtime "PYTHON:3.12" --sku B1
az webapp config set -g rg-kundcase -n kundcase-mindcamp --startup-file "$(cat startup.txt)"
az webapp config appsettings set -g rg-kundcase -n kundcase-mindcamp --settings \
  DATABASE_URL='postgresql+psycopg://kundcaseadmin:<lösenord>@pg-kundcase-mindcamp.postgres.database.azure.com:5432/kundcase?sslmode=require' \
  ANTHROPIC_API_KEY='<nyckel>' \
  BASE_URL='https://kundcase-mindcamp.azurewebsites.net' \
  CLAUDE_MODEL='claude-sonnet-5-5' \
  SCM_DO_BUILD_DURING_DEPLOYMENT=true
```
`DEV_USER` sätts inte.

- [ ] **Steg 4: Slå på inloggning (portalen)**

App Service → Authentication → Add identity provider → Microsoft:
- Tenant: Mindcamps tenant (single tenant)
- "Unauthenticated requests": **Allow unauthenticated access** (kundsidan måste vara öppen; `/admin` skyddas av appen)

- [ ] **Steg 5: Verifiera**

1. `https://kundcase-mindcamp.azurewebsites.net/admin` i privat fönster → "Inte inloggad" med inloggningslänk.
2. Logga in → listan visas med din e-post.
3. Kör samma manuella flöde som i Task 9.

- [ ] **Steg 6: Commit**

```bash
git add startup.txt
git commit -m "Startkommando för Azure App Service"
git push
```
