# Indexing Wikipedia with Elasticsearch

## Objective
Take 14GiB (compressed) of Wikipedia data and make it searchable

## What is Elasticsearch?

* A database
* A distributed database
* A distributed database that's good at searching human language
* A distributed database that's good at searching human language and aggregation

## How Elasticsearch Stores Documents

* It stores **documents** on a **node**(1 instance)
* Multiple nodes form a **cluster**
* There are more schema details, but they aren't important for this talk

## Lucene

If you know what Lucene is, Elasticsearch is based on it. If you don't, you can ignore this.

## Back to the Search

## Desired Search Properties

* Relevance (returned documents should be good matches for the query)
* Speed (queries should return quickly)
* Statistics (we might want some aggregate stats about the Wiki corpus)

## Steps

* Read Wikipedia Data
* Coerce Data into a Useful Document Schema
* Load Documents into Database
* Perform a Search

## What Data are We Working With?

* Titles
* Redirects (an alias for a title)
* Body Text (plus rich markup)

## How do we Define Success?

* A search for an exact article title turns it up as #1
* A search for something missing a word or two still finds it
* A search for a general idea or related group of words should find the most similar article.

## How we Frame the Problem

For search problems we think in terms of **relevance** or **scoring** instead of
**boolean** matching.

```sql
--- Boolean query, does it have either term?
SELECT * FROM users WHERE hobbies LIKE '%basketball%' OR hobbies LIKE '%baseball%'
```

```sql
--- Boolean query with a (poor) rank algorithm
SELECT
username, hobbies
CASE
  WHEN hobbies LIKE '%basketball%'
  THEN 1
  WHEN hobbies LIKE '%baseball%'
  THEN 1
  WHEN hobbies LIKE '%baseball%' AND hobbies LIKE '%basketball%'
  THEN 2
END AS score
FROM users
ORDER BY score DESC
```

## Elasticsearch/Lucene are Built for This

Elasticsearch can answer questions like:
"Find me users named 'Pat', tolerating a misspelling of up to two characters,
 also, make sure these users have 'basketball' listed among their hobbies within
 up to three words of the term 'coach' or 'coaching', or any other 'coach'-like words,
 ranking these results weighted toward how closely their username matches, but also by how frequently
 the text sees 'basketball' and coach around each other. Also, while you're at it,
 give me a histogram of the 5 year age groups these users fall into."

```json
{
  "query": {
    "query_string": "username:\"Pat\"~ AND hobbies:\"basketball coach\"~3"
  },
  "aggs": {"ages": {"histogram": {"field": "age", "interval": 5}}}
}
```

## Let's Create a Simple Wikipedia Search Engine

* If the title or a redirect exactly matches the query make that our #1 result
* If the title is very similar to the query it should be given a very high score
* If the body contains the query with disproportionate frequency that should boost the document's score

## Here's a sample document we'll match against

```json
{
  "title": "United States",
  "redirects": ["USA", "United States of America"],
  "text": "The United States of America (USA), commonly referred to..."
}
```

## The Essence of Our Query

```javascript
// Match the title OR a redirect exactly, give this a mega-boost
title.title_simple:"{{qs}}"^10000
OR redirects.redirects_simple:"{{qs}}"^10000
// Match the analyzed title or redirects, give this a big boost
OR title.title_snow:"{{qs}}"^100
OR redirects.redirects_snow:"{{qs}}"^100
// Match the text of the body either with simple analysis or advanced
// If it matches the simpler analysis exactly give it a small boost
OR body.body_simple:"{{qs}}"^2
OR body.body_snow:"{{qs}}"
```

## Here's Our Query

```json
POST /en-wikipedia2/_search
{
  "fields": ["title"],
  "highlight": {
    "fields": {"title": {}, "redirects": {},"body": {}}
  },
  "query": {
    "query_string": {
      "query": "title.title_simple:\"united States\"^10000 OR redirects.redirects_simple:\"united States\"^1000 OR title.title_snow:\"united States\"^10 OR redirects.redirects_snow:\"united States\"^10 OR body.body_simple:\"united States\"^5 OR body.body_snow:\"united States\""
    }
  }
}
```

## Making it Searchable

* We need to **analyze** the text.
* That means creating representation of the text that maximizes the machine's understanding of it.

## Stemming

![Analysis]('img/analysis.png')

## What is Analysis?

Analysis == Turning a string into tokens
Turn 'Wonderous "Book of Secrets" finds home in secret books of wonder'
into:

```
# Analyzed with snowball filter
['wonder', '', 'book', 'secret', 'find', 'home', '', 'secret', 'book', 'wonder']
```

[Example](http://localhost:9200/en-wikipedia2/_analyze?analyzer=snowball&text=Book%20of%20Secrets%20finds%20home%20at%20in%20secret%20books%20of%20wonder&pretty=true)

## Why Does this Make Search Fast / Accurate?

When you analyze both the query and the text searches become simple lookups or use
other efficient techniques. The analyzed text is stored in an extremely efficient to search
internal representation.

## Which Analyzers To Use?

* **Snowball** This analyzer is great for stemming English words.
* **Simple** Split the text on whitespace, then lowercase it

## Here's Our Query

```json
POST /en-wikipedia2/_search
{
  "fields": ["title"],
  "highlight": {
    "fields": {"title": {}, "redirects": {},"body": {}}
  },
  "query": {
    "query_string": {
      "query": "title.title_simple:\"united States\"^10000 OR redirects.redirects_simple:\"united States\"^1000 OR title.title_snow:\"united States\"^10 OR redirects.redirects_snow:\"united States\"^10 OR body.body_simple:\"united States\"^5 OR body.body_snow:\"united States\""
    }
  }
}
```

## Some Fun Tricks

## Get the Most Common Words in Titles

```json
GET /en-wikipedia2/_search?search_type=count
{
  "fields": ["title"],
  "aggregations" : {
    "topterms" : {
      "terms" : { "field" : "title.title_simple", "size": 100 }
    }
  }
}
```

## Perform a Fast Autocomplete

```json
# Auto-complete suggest
GET en-wikipedia2/_suggest
{
  "page-suggest": {
    "text": "Futuram",
    "completion": {"field": "suggest"}
  }
}
```

## Get the United States article's MLT docs

```json
GET en-wikipedia2/_search
{
  "fields": ["title"],
  "query": {
    "more_like_this": {
      "fields": ["title.title_snow", "redirects.redirects_snow", "body.body_snow"],
      "docs": [{
        "_type": "page",
        "_id": "AU5lmkRiwqDJyRWGOM94"
      }]
    }
  }
}
```


# Get the United States Environmental Protection Agency article's MLT docs
```json
GET en-wikipedia2/_search
{
  "fields": ["title"],
  "query": {
    "more_like_this": {
      "fields": ["title.title_snow", "redirects.redirects_snow", "body.body_snow"],
      "docs": [{
        "_type": "page",
        "_id": "AU5lhtPiwqDJyRWGLSDt"
      }]
    }
  }
}
```

# More Goodies

* Geosearch
* Numeric Processing
* Weighting by popularity or page rank
