
#frontend
import requests
from bs4 import BeautifulSoup
import sqlite3

# URLs for CDP documentation
DOC_URLS = {
    "segment": "https://segment.com/docs/",
    "mparticle": "https://docs.mparticle.com/",
    "lytics": "https://docs.lytics.com/",
    "zeotap": "https://docs.zeotap.com/home/en-us/"
}

# SQLite database for storing indexed content
DB_NAME = "cdp_docs.db"

def scrape_and_store_docs():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS docs (id INTEGER PRIMARY KEY, platform TEXT, url TEXT, content TEXT)")
    
    for platform, url in DOC_URLS.items():
        response = requests.get(url)
        soup = BeautifulSoup(response.text, 'html.parser')
        sections = soup.find_all('a')  # Assuming links are the main navigational elements
        
        for section in sections:
            link = section.get('href')
            if link and link.startswith("http"):
                page_response = requests.get(link)
                page_soup = BeautifulSoup(page_response.text, 'html.parser')
                content = page_soup.get_text(separator="\n")
                cursor.execute("INSERT INTO docs (platform, url, content) VALUES (?, ?, ?)", (platform, link, content))
    
    conn.commit()
    conn.close()

if _name_ == "_main_":
    scrape_and_store_docs()
    #backend
    from flask import Flask, request, jsonify
import sqlite3
from whoosh.index import create_in
from whoosh.qparser import QueryParser
from whoosh.fields import Schema, TEXT, ID

app = Flask(_name_)
DB_NAME = "cdp_docs.db"
INDEX_DIR = "indexdir"

# Create Whoosh schema and index
def create_index():
    schema = Schema(platform=TEXT(stored=True), url=ID(stored=True), content=TEXT)
    index = create_in(INDEX_DIR, schema)
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT platform, url, content FROM docs")
    
    writer = index.writer()
    for platform, url, content in cursor.fetchall():
        writer.add_document(platform=platform, url=url, content=content)
    writer.commit()
    conn.close()

@app.route("/query", methods=["POST"])
def query_docs():
    user_query = request.json.get("query", "")
    platform = request.json.get("platform", "").lower()
    
    from whoosh.index import open_dir
    index = open_dir(INDEX_DIR)
    searcher = index.searcher()
    query_parser = QueryParser("content", index.schema)
    query = query_parser.parse(user_query)
    results = searcher.search(query, limit=5)
    
    response = [{"platform": r["platform"], "url": r["url"], "content": r.highlights("content")} for r in results]
    return jsonify(response)

if _name_ == "_main_":
    create_index()  # Ensure index is created
    app.run(debug=True)
    #testing
    import pytest
import requests

BASE_URL = "http://localhost:5000"

def test_query_endpoint():
    payload = {"query": "How to create a source?", "platform": "segment"}
    response = requests.post(f"{BASE_URL}/query", json=payload)
    assert response.status_code == 200
    assert isinstance(response.json(), list)
    
