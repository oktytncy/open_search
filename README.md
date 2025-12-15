### Create a Virtual Environment [Optional]

Some environments can be managed externally by a system package manager, such as Homebrew in macOS. 

This means that your system may prevent you from installing packages globally to avoid conflicts with packages managed by the system package manager. This may pose an obstacle to package installation.

**To avoid this issue, you can Create a Virtual Environment before running python or pip commands.**

```bash
cd "<YOUR-INSTALLATION-PATH>"
```

#### on Linux/Mac/..

Install environment and activate the virtual environment.

```python
python3.11 -m venv myenv
```

```bash
source myenv/bin/activate
```

## Set Up Development OpenSearch Cluster

1. Create a directory for your OpenSearch cluster:

    ```shell
    mkdir opensearch-cluster
    cd opensearch-cluster
    ```

2. Create the Docker Compose file

    > In this directory, create `docker-compose.yml` and paste the development compose contents into it:

    ```shell
    wget -O docker-compose.yml https://raw.githubusercontent.com/oktytncy/open_search/main/manifests/docker-compose.yml
    ```

3. Install Docker Compose

    ```shell
    brew install docker-compose
    ```

4. Verify with the following command.

    ```
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
    ```

    💡 Expected Output:
    ```shell
    (myenv) (base) oktay.tuncay@Oktay-Tuncay-H76R2P9W6H open_search % docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

    NAMES                   STATUS         PORTS
    opensearch-dashboards   Up 2 minutes   0.0.0.0:5601->5601/tcp, [::]:5601->5601/tcp
    opensearch-node1        Up 2 minutes   0.0.0.0:9200->9200/tcp, [::]:9200->9200/tcp, 0.0.0.0:9600->9600/tcp, [::]:9600->9600/tcp
    opensearch-node2        Up 2 minutes   9200/tcp, 9300/tcp, 9600/tcp, 9650/tcp
    ```

4. Confirm OpenSearch responds.

    ```shell
    curl -s http://localhost:9200/ | head
    ```


Port 5601 is the default web UI port for OpenSearch Dashboards (and historically for Kibana, which OpenSearch Dashboards was forked from).

3. Access OpenSearch Dashboards

    > OpenSearch Dashboards is the web-based UI for OpenSearch. It runs on port **5601** by default (similar to Kibana).
    > 
    > Once the containers are running, open your browser and go to: http://localhost:5601

    > 📌 Conceptually:
    > 	- 9200 -> OpenSearch API
    > 		- Used by apps, curl, integrations
    > 		- REST/JSON interface
    > 	- 5601 -> OpenSearch Dashboards (Web UI)
    > 		- Used by humans in the browser
    > 		- Visualizations, dashboards, Dev Tools
    > So they’re intentionally separated:
    > 	- Machines -> 9200
    > 	- Humans -> 5601

    > You may see a popup about a new **Enhanced Discover experience**. This is optional and can be safely dismissed for local or development setups.

## OpenSearch Dashboards: Dev Tools + First API Calls

### Dev Tools: the **Console** inside Dashboards

> Go to **☰ Menu** -> Management -> Dev Tools

Dev Tools is basically **curl, but inside the UI**. You paste REST requests (GET/PUT/POST/DELETE) and see JSON responses.

1. Check your OpenSearch is healthy.

    ```console
    GET _cluster/health
    ```

    > ---
    > 👀 What to look for:
    > - status: green -> perfect
    > - status: yellow -> usually OK for local learning (often happens when replicas exist but you don’t have enough nodes)
    > - status: red -> something is wrong (don’t continue until fixed)
    >
    > ---

    <p align="middle">
        <img src="images/1.png" alt="drawing" width="900"/>
    </p>

    > ---
    > 📌 Other useful commands:
    >   - See nodes:
    >       - GET _cat/nodes?v
    >   - See indices (tables)
    >       - GET _cat/indices?v
    > 
    > If this is a fresh setup, you might see few or no indices. That’s fine.
    >
    > ---

### Build a StreamFlix dataset (catalog + users + watch events)

> **Why this dataset**
> It lets you practice the most useful OpenSearch concepts in one story:
> - **Search titles like a user** (full-text search)
> - **Synonyms** (“sci-fi” ≈ “science fiction”, “romcom” ≈ “romantic comedy”)
> - **Autocomplete** (typing “orbi…” suggests “Orbital Heist”)
> - **Nested fields** (cast members with role + name matched together)
> - **Analytics** (top genres, watch time by device, watch activity over time)

#### Create the indexes

##### Titles index: search + synonyms + autocomplete + nested cast

First, lets design the `schema` and `text rules` for the movie/series catalog.

In OpenSearch, you don’t just dump JSON and hope for the best. You first decide:

1. What fields exist (title, description, genres, release_date, cast…)
2. How text should be understood (normal search, synonyms, autocomplete)
3. How arrays of objects should behave

##### Step 1: Create a basic Titles index (no synonyms, no autocomplete yet)

Let’s start simple and create an index that supports normal search + correct cast structure.

> Run the below script in Dev Tools:

```console
PUT streamflix_titles
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "title_id":     { "type": "keyword" },
      "title":        { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "description":  { "type": "text" },
      "type":         { "type": "keyword" },
      "genres":       { "type": "keyword" },
      "release_date": { "type": "date" },
      "runtime_min":  { "type": "integer" },
      "rating":       { "type": "float" },
      "languages":    { "type": "keyword" },
      "cast": {
        "type": "nested",
        "properties": {
          "name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
          "role": { "type": "keyword" }
        }
      }
    }
  }
}
```

✅ What you should see:

```console
"acknowledged": true
```

Then verify count:

```console
GET streamflix_titles/_count
```

<p align="middle">
    <img src="images/2.png" alt="drawing" width="900"/>
</p>

##### Step 2 — Insert 3 titles

Run these in Dev Tools one by one:

###### 2.1 Add title 1

```console
POST streamflix_titles/_doc/T-001
{
  "title_id": "T-001",
  "title": "Orbital Heist",
  "description": "A science fiction crew plans a heist in low orbit.",
  "type": "movie",
  "genres": ["sci-fi","thriller"],
  "release_date": "2024-09-12",
  "runtime_min": 116,
  "rating": 4.4,
  "languages": ["en"],
  "cast": [
    {"name":"Mina Kaya","role":"Captain"},
    {"name":"Jonas Reed","role":"Engineer"}
  ]
}
```

###### Add title 2

```console
POST streamflix_titles/_doc/T-002
{
  "title_id": "T-002",
  "title": "Canal Nights",
  "description": "A romantic comedy set in Amsterdam—bikes, rain, and one missed tram.",
  "type": "movie",
  "genres": ["romcom"],
  "release_date": "2023-02-10",
  "runtime_min": 98,
  "rating": 4.0,
  "languages": ["en","nl"],
  "cast": [
    {"name":"Elise van Dijk","role":"Lead"},
    {"name":"Marco Silva","role":"Lead"}
  ]
}
```

###### Add title 3

```console
POST streamflix_titles/_doc/T-003
{
  "title_id": "T-003",
  "title": "Shadow Protocol",
  "description": "A suspense thriller about a leaked key, a false alarm, and a chase.",
  "type": "series",
  "genres": ["thriller"],
  "release_date": "2025-01-14",
  "runtime_min": 50,
  "rating": 4.1,
  "languages": ["en"],
  "cast": [
    {"name":"Mina Kaya","role":"Detective"},
    {"name":"Ari Patel","role":"Analyst"}
  ]
}
```

###### Confirm you have 3 docs

```console
GET streamflix_titles/_count
```

##### Search in description

```console
GET streamflix_titles/_search
{
  "query": {
    "match": {
      "description": "science fiction heist"
    }
  }
}
```

<p align="middle">
    <img src="images/3.png" alt="drawing" width="900"/>
</p>

> ---
> What you should see 🔦
> - hits.total.value should be >= 1
> - Orbital Heist should be near the top
>
> ---

- You used:
  
  ```console
  "match": { "description": "science fiction heist" }
  ```
**Meaning:** `Find documents where the meaning/words match.` OpenSearch breaks text into tokens and scores results (_score).

##### Exact match using

Your title field is text, but OpenSearch also created title.keyword (exact version).

```console
GET streamflix_titles/_search
{
  "_source": ["title_id","title"],
  "query": {
    "term": { "title.keyword": "Orbital Heist" }
  }
}
```

- **Expected:** 1 hit.

Now try a tiny change (lowercase):

```console
GET streamflix_titles/_search
{
  "_source": ["title_id","title"],
  "query": {
    "term": { "title.keyword": "orbital heist" }
  }
}
```

- **Expected:** 0 hits - because keyword is exact + case-sensitive.

Text match is more forgiving (match)

```console
GET streamflix_titles/_search
{
  "_source": ["title_id","title"],
  "query": {
    "match": { "title": "orbital heist" }
  }
}
```

- **Expected:** It should find Orbital Heist even with different casing.

##### Filters (no scoring, just rules)

Filters are useful for things like genre/type/date.

```console
GET streamflix_titles/_search
{
  "_source": ["title_id","title","type","genres","release_date"],
  "query": {
    "bool": {
      "filter": [
        { "term":  { "type": "movie" } },
        { "term":  { "genres": "thriller" } },
        { "range": { "release_date": { "gte": "2024-01-01" } } }
      ]
    }
  }
}
```

**Meaning:** `Give me titles that match all these conditions.`

##### Why nested for cast

Because cast is an array of objects (name + role), and we want `name + role` to match inside the same cast entry.


```console
GET streamflix_titles/_search
{
  "_source": ["title_id","title","cast"],
  "query": {
    "nested": {
      "path": "cast",
      "query": {
        "bool": {
          "must": [
            { "term": { "cast.name.keyword": "Mina Kaya" } },
            { "term": { "cast.role": "Detective" } }
          ]
        }
      }
    }
  }
}
```

- **Expected:** It should return the title where Mina Kaya is actually a Detective.