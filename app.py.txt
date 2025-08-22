from __future__ import annotations
import sys, time, hashlib, datetime as dt, os
from dataclasses import dataclass
from typing import List, Optional

import requests
from bs4 import BeautifulSoup
from sqlmodel import SQLModel, Field, Session, create_engine, select
from flask import Flask, render_template_string, request

# ----------------------
# 1) Konfiguracija
# ----------------------
@dataclass
class Config:
    SOURCE_URL: str = os.getenv("SOURCE_URL", "https://example.com/tenders")
    ITEM_SELECTOR: str = ".tender-item"
    TITLE_SELECTOR: str = ".tender-title"
    LINK_SELECTOR: str = "a"
    DATE_SELECTOR: str = ".tender-date"
    HEADERS: dict = None
    TIMEOUT: int = 20

CFG = Config(HEADERS={"User-Agent": "MiniScraper/1.0"})

# ----------------------
# 2) Model i baza
# ----------------------
class Record(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    title: str
    url: str
    published_at: Optional[dt.date] = None
    source: str
    url_hash: str = Field(index=True, unique=True)
    created_at: dt.datetime = Field(default_factory=lambda: dt.datetime.utcnow())

DB_URL = os.getenv("DATABASE_URL", "sqlite:///scraper.db")
if DB_URL.startswith("postgres://"):
    DB_URL = DB_URL.replace("postgres://", "postgresql://", 1)
ENGINE = create_engine(DB_URL, echo=False)
SQLModel.metadata.create_all(ENGINE)

# ----------------------
# 3) Scraper
# ----------------------
MOCK_HTML = """
<html><body>
<div class="tender-item"><a href="https://demo/1" class="tender-title">Natječaj 1</a><div class="tender-date">2025-08-01</div></div>
<div class="tender-item"><a href="https://demo/2" class="tender-title">Natječaj 2</a><div class="tender-date">2025-08-18</div></div>
</body></html>
"""

def sha256(text: str) -> str: return hashlib.sha256(text.encode()).hexdigest()

def parse_date(text: str) -> Optional[dt.date]:
    try: return dt.date.fromisoformat(text.strip())
    except Exception: return None

def fetch_html(url: str) -> str:
    try:
        r = requests.get(url, headers=CFG.HEADERS, timeout=CFG.TIMEOUT)
        if r.status_code == 200 and r.text.strip(): return r.text
    except Exception: pass
    return MOCK_HTML

def scrape_once(url: str) -> List[Record]:
    soup = BeautifulSoup(fetch_html(url), "html.parser")
    items = soup.select(CFG.ITEM_SELECTOR)
    results: List[Record] = []
    for it in items:
        title_el, link_el, date_el = it.select_one(CFG.TITLE_SELECTOR), it.select_one(CFG.LINK_SELECTOR), it.select_one(CFG.DATE_SELECTOR)
        title = title_el.get_text(strip=True) if title_el else "(bez naslova)"
        href = (link_el.get("href") if link_el else "")
        date_val = parse_date(date_el.get_text(strip=True)) if date_el else dt.date.today()
        results.append(Record(title=title, url=href, published_at=date_val, source=url, url_hash=sha256(href or title)))
    return results

def upsert_records(recs: List[Record]) -> int:
    new_count=0
    with Session(ENGINE) as ses:
        for r in recs:
            exists = ses.exec(select(Record).where(Record.url_hash==r.url_hash)).first()
            if exists:
                if r.title!=exists.title: exists.title=r.title; ses.add(exists)
            else:
                ses.add(r); new_count+=1
        ses.commit()
    return new_count

# ----------------------
# 4) Flask app
# ----------------------
app = Flask(__name__)
BASE_TMPL = """<h1>Mini Scraper</h1><a href='/scrape-now'>Scrape</a><table>{% for r in records %}<tr><td>{{r.title}}</td><td>{{r.published_at}}</td><td><a href='{{r.url}}'>link</a></td></tr>{% endfor %}</table>"""

@app.route("/")
def index():
    q=(request.args.get("q") or "").lower()
    with Session(ENGINE) as ses:
        records=list(ses.exec(select(Record)).all())
    if q: records=[r for r in records if q in (r.title or "").lower()]
    return render_template_string(BASE_TMPL,records=records)

@app.route("/healthz")
def healthz(): return "ok",200

@app.route("/scrape-now")
def scrape_now():
    token=os.getenv("SCRAPE_TOKEN"); qtoken=request.args.get("token")
    if token and qtoken!=token: return "unauthorized",401
    new_count=upsert_records(scrape_once(CFG.SOURCE_URL))
    return f"Dodano {new_count} novih zapisa. <a href='/'>&larr; back</a>"

# ----------------------
# 5) CLI entry
# ----------------------
def run_scrape():
    recs=scrape_once(CFG.SOURCE_URL); added=upsert_records(recs)
    print(f"Pronađeno {len(recs)}, dodano {added}")

def run_server(): app.run(debug=True)

if __name__=="__main__":
    if len(sys.argv)>1:
        if sys.argv[1]=="scrape": run_scrape()
        elif sys.argv[1]=="run": run_server()
        else: print("Use scrape|run")
    else: print("Usage: python app.py [scrape|run]")
