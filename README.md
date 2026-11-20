"""Auto news channel: reads RSS feeds, has Claude rewrite each item in original
words, and builds docs/index.html with source credit on every story."""
import json, os, re, urllib.request, xml.etree.ElementTree as ET
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

API_KEY = os.environ["ANTHROPIC_API_KEY"]
MODEL = "claude-haiku-5-5"   # cheapest current model
PER_FEED = 3                 # new stories per feed per run (lower = cheaper)
KEEP = 300                   # max stories kept on the page

# (Category, Source name, RSS URL). Add or remove lines freely.
# Check each outlet's terms of use before publishing publicly.
FEEDS = [
    ("World", "BBC News", "https://feeds.bbci.co.uk/news/world/rss.xml"),
    ("World", "NPR", "https://feeds.npr.org/1004/rss.xml"),
    ("US/National", "NPR", "https://feeds.npr.org/1003/rss.xml"),
    ("Business", "BBC News", "https://feeds.bbci.co.uk/news/business/rss.xml"),
    ("Tech", "BBC News", "https://feeds.bbci.co.uk/news/technology/rss.xml"),
    ("Tech", "Ars Technica", "https://feeds.arstechnica.com/arstechnica/index"),
    ("Science", "BBC News", "https://feeds.bbci.co.uk/news/science_and_environment/rss.xml"),
    ("Health", "BBC News", "https://feeds.bbci.co.uk/news/health/rss.xml"),
    ("Sports", "BBC Sport", "https://feeds.bbci.co.uk/sport/rss.xml"),
    ("Entertainment", "BBC News", "https://feeds.bbci.co.uk/news/entertainment_and_arts/rss.xml"),
    ("Space", "NASA", "https://www.nasa.gov/feed/"),
    ("Culture", "The Guardian", "https://www.theguardian.com/culture/rss"),
    ("Travel", "The Guardian", "https://www.theguardian.com/travel/rss"),
    ("Food", "The Guardian", "https://www.theguardian.com/food/rss"),
    ("Environment", "The Guardian", "https://www.theguardian.com/environment/rss"),
]

def get(url):
    req = urllib.request.Request(url, headers={"User-Agent": "NewsChannelBot/1.0"})
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.read()

def clean(t):
    return re.sub(r"\s+", " ", re.sub(r"<[^>]+>", " ", t or "")).strip()

def read_feed(url):
    root = ET.fromstring(get(url))
    for it in root.iter("item"):
        title, link = clean(it.findtext("title")), (it.findtext("link") or "").strip()
        desc = clean(it.findtext("description"))
        try:
            d = parsedate_to_datetime(it.findtext("pubDate")).date().isoformat()
        except Exception:
            d = datetime.now(timezone.utc).date().isoformat()
        if title and link.startswith("http"):
            yield title, desc, link, d

def rewrite(title, desc, source):
    prompt = (
        f"Source: {source}\nOriginal headline: {title}\nOriginal snippet: {desc}\n\n"
        "Write a NEW headline and a 2-sentence summary in completely original wording "
        "and sentence structure. Use ONLY facts stated above; add nothing, guess nothing. "
        "Do not reuse phrases from the original. Reply with JSON only: "
        '{"headline": "...", "summary": "..."}'
    )
    body = json.dumps({"model": MODEL, "max_tokens": 400,
                       "messages": [{"role": "user", "content": prompt}]}).encode()
    req = urllib.request.Request(
        "https://api.anthropic.com/v1/messages", data=body,
        headers={"x-api-key": API_KEY, "anthropic-version": "2023-06-01",
                 "content-type": "application/json"})
    with urllib.request.urlopen(req, timeout=60) as r:
        text = json.loads(r.read())["content"][0]["text"]
    return json.loads(re.search(r"\{.*\}", text, re.S).group(0))

def main():
    path = "docs/stories.json"
    stories = json.load(open(path)) if os.path.exists(path) else []
    seen = {s["link"] for s in stories}
    new = []
    for cat, source, url in FEEDS:
        try:
            items = [i for i in read_feed(url) if i[2] not in seen][:PER_FEED]
        except Exception as e:
            print("Feed failed:", url, e); continue
        for title, desc, link, d in items:
            if len(desc) < 40:       # too little info to summarize safely
                continue
            try:
                r = rewrite(title, desc, source)
                new.append({"headline": r["headline"], "summary": r["summary"],
                            "category": cat, "source": source, "link": link, "date": d})
                seen.add(link)
            except Exception as e:
                print("Rewrite failed:", link, e)
    stories = (new + stories)[:KEEP]
    os.makedirs("docs", exist_ok=True)
    json.dump(stories, open(path, "w"), indent=1)
    data = json.dumps(stories).replace("</", "<\\/")
    open("docs/index.html", "w").write(PAGE.replace("__DATA__", data))
    print(f"Added {len(new)} stories, {len(stories)} total.")

PAGE = """<!DOCTYPE html><html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1"><title>My News Channel</title>
<style>
:root{--bg:#f6f4ef;--card:#fff;--text:#1c1a17;--muted:#6a655d;--accent:#b3321f;--line:#e3ded4}
@media(prefers-color-scheme:dark){:root{--bg:#171513;--card:#232020;--text:#eeeae3;--muted:#a39d93;--accent:#ef7a63;--line:#3a3632}}
body{margin:0;background:var(--bg);color:var(--text);font:16px/1.5 Georgia,serif}
main{max-width:600px;margin:0 auto;padding:16px 14px}h1{margin:0}
.s{color:var(--muted);font-size:.9rem}
.chips{display:flex;gap:6px;overflow-x:auto;padding:8px 0}
.chip{flex:none;font:inherit;background:var(--card);color:var(--text);border:1px solid var(--line);border-radius:99px;padding:4px 12px}
.chip.on{background:var(--accent);color:#fff;border-color:var(--accent)}
.card{background:var(--card);border:1px solid var(--line);border-radius:12px;padding:14px;margin-bottom:10px}
.tag{font-size:.72rem;text-transform:uppercase;letter-spacing:.06em;color:var(--accent);font-weight:bold}
.card h3{margin:2px 0 6px;font-size:1.1rem}.card p{margin:0 0 8px}
.cred{font-size:.85rem;color:var(--muted);border-top:1px solid var(--line);padding-top:8px}.cred a{color:var(--accent)}
</style></head><body><main>
<h1>My News Channel</h1>
<p class="s">Summaries are AI-written from headlines and snippets. Always read the original for the full story.</p>
<div class="chips" id="chips"></div><div id="list"></div></main>
<script>
const S=__DATA__;let cat="All";
const cats=["All",...new Set(S.map(s=>s.category))];
function chips(){const c=document.getElementById("chips");c.innerHTML="";cats.forEach(x=>{const b=document.createElement("button");b.className="chip"+(x===cat?" on":"");b.textContent=x;b.onclick=()=>{cat=x;chips();render()};c.appendChild(b)})}
function render(){const L=document.getElementById("list");L.innerHTML="";
S.filter(s=>cat==="All"||s.category===cat).forEach(s=>{
const d=document.createElement("div");d.className="card";
const t=document.createElement("div");t.className="tag";t.textContent=s.category;
const h=document.createElement("h3");h.textContent=s.headline;
const p=document.createElement("p");p.textContent=s.summary;
const c=document.createElement("div");c.className="cred";c.append("Source: "+s.source+" \\u00b7 "+s.date+" \\u00b7 ");
const a=document.createElement("a");a.textContent="Read original";a.href=s.link;a.target="_blank";a.rel="noopener";c.appendChild(a);
d.append(t,h,p,c);L.appendChild(d)})}
chips();render();
</script></body></html>"""

if __name__ == "__main__":
    main()