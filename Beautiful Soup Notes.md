# 🌿 Beautiful Soup Notes (Beginner Friendly)

## What is Beautiful Soup?

Beautiful Soup is a Python library used for reading and extracting data from HTML pages. It makes it easy to scrape information from websites.

---

## ⭐ Step 1: Installation

```bash
pip install beautifulsoup4
```

---

## ⭐ Step 2: Basic Example

```python
from bs4 import BeautifulSoup

html_code = """
<html>
    <body>
        <h1>Hello World</h1>
        <p class="info">This is a paragraph.</p>
    </body>
</html>
"""

soup = BeautifulSoup(html_code, "html.parser")

title_tag = soup.find("h1")
paragraph_tag = soup.find("p", class_="info")

print(title_tag.text)
print(paragraph_tag.text)
```

### Explanation

* `from bs4 import BeautifulSoup` → Import the library.
* `html_code` → Sample HTML.
* `soup` → Parses the HTML.
* `find("h1")` → Finds the first `<h1>`.
* `find("p", class_="info")` → Finds a `<p>` with class `info`.
* `.text` → Extracts only the text.

---

## ⭐ Step 3: Scraping a Real Website

```python
import requests
from bs4 import BeautifulSoup

url = "http://quotes.toscrape.com"
response = requests.get(url)

soup = BeautifulSoup(response.text, "html.parser")

quotes = soup.find_all("span", class_="text")

for q in quotes:
    print(q.text)
```

### Explanation

* `requests.get()` → Downloads the webpage.
* `find_all()` → Finds all matching tags.
* Loop prints each quote.

---

## ⭐ Extracting Links

```python
for link in soup.find_all("a"):
    print(link.get("href"))
```

---

## ⭐ Extracting Image Sources

```python
img = soup.find("img")
print(img.get("src"))
```

---

## ⭐ CSS Selectors

```python
titles = soup.select("div.quote span.text")
```

---

## Tips

* Check if a website allows scraping.
* Use delays when scraping many pages.
* Beautiful Soup only reads HTML; it doesn’t load pages on its own.
