# Movie Recommendation

## Environment setup

Create and activate the project virtual environment, then install the dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Copy `.env.example` to `.env` and fill in your credentials. The notebook loads the
values with `python-dotenv`; `.env` is ignored by Git and should never be committed.
In VS Code, select the `.venv` Python interpreter/kernel before running the notebook.

Required variables:

- `GROQ_API_KEY`
- `NEO4J_URI` (Neo4j Bolt URI, such as `neo4j+s://<instance>.databases.neo4j.io`)
- `NEO4J_USERNAME`
- `NEO4J_PASSWORD`

## Movie graph data

The movie recommender expects this graph structure:

- Nodes: `Person` (`name`), `Movie` (`title`, `released`), and `Genre` (`name`)
- Relationships: `(Person)-[:ACTED_IN]->(Movie)`,
	`(Person)-[:DIRECTED]->(Movie)`, and `(Movie)-[:IN_GENRE]->(Genre)`

### Uniqueness constraints

Create these constraints before loading the sample data. They ensure each person,
movie, and genre is uniquely identified by its name or title, and support the
`MERGE` operations below. If the database already contains duplicate values for
these properties, remove or reconcile those duplicates before creating constraints.

```cypher
CREATE CONSTRAINT person_name_unique IF NOT EXISTS
FOR (p:Person)
REQUIRE p.name IS UNIQUE;

CREATE CONSTRAINT movie_title_unique IF NOT EXISTS
FOR (m:Movie)
REQUIRE m.title IS UNIQUE;

CREATE CONSTRAINT genre_name_unique IF NOT EXISTS
FOR (g:Genre)
REQUIRE g.name IS UNIQUE;
```

Run the following statements in order in Neo4j Browser or the Neo4j Aura Query
editor to create the sample data. They use `MERGE`, so they can be run again
without creating duplicate nodes or relationships. The people list includes every
person referenced by the relationship data so all `MATCH` clauses can succeed.

### 1. Create people

```cypher
UNWIND [
	{name: "Leonardo DiCaprio"},
	{name: "Joseph Gordon-Levitt"},
	{name: "Tom Hardy"},
	{name: "Christian Bale"},
	{name: "Cillian Murphy"},
	{name: "Matthew McConaughey"},
	{name: "Christopher Nolan"},
	{name: "Alejandro G. Iñárritu"}
] AS person
MERGE (:Person {name: person.name});
```

### 2. Create movies

```cypher
UNWIND [
	{title: "Inception", released: 2010},
	{title: "The Dark Knight Rises", released: 2012},
	{title: "Interstellar", released: 2014},
	{title: "Dunkirk", released: 2017},
	{title: "The Revenant", released: 2015},
	{title: "The Dark Knight", released: 2008}
] AS movieData
MERGE (m:Movie {title: movieData.title})
SET m.released = movieData.released;
```

### 3. Connect actors to movies

```cypher
UNWIND [
	{person: "Leonardo DiCaprio", movie: "Inception"},
	{person: "Joseph Gordon-Levitt", movie: "Inception"},
	{person: "Tom Hardy", movie: "Inception"},
	{person: "Christian Bale", movie: "The Dark Knight"},
	{person: "Christian Bale", movie: "The Dark Knight Rises"},
	{person: "Tom Hardy", movie: "The Dark Knight Rises"},
	{person: "Cillian Murphy", movie: "The Dark Knight"},
	{person: "Cillian Murphy", movie: "Inception"},
	{person: "Leonardo DiCaprio", movie: "The Revenant"},
	{person: "Tom Hardy", movie: "The Revenant"},
	{person: "Matthew McConaughey", movie: "Interstellar"},
	{person: "Cillian Murphy", movie: "Dunkirk"}
] AS item
MATCH (p:Person {name: item.person})
MATCH (m:Movie {title: item.movie})
MERGE (p)-[:ACTED_IN]->(m);
```

### 4. Connect directors to movies

```cypher
UNWIND [
	{person: "Christopher Nolan", movie: "Inception"},
	{person: "Christopher Nolan", movie: "The Dark Knight"},
	{person: "Christopher Nolan", movie: "The Dark Knight Rises"},
	{person: "Christopher Nolan", movie: "Interstellar"},
	{person: "Christopher Nolan", movie: "Dunkirk"},
	{person: "Alejandro G. Iñárritu", movie: "The Revenant"}
] AS item
MATCH (d:Person {name: item.person})
MATCH (m:Movie {title: item.movie})
MERGE (d)-[:DIRECTED]->(m);
```

### 5. Connect movies to genres

```cypher
UNWIND [
	{movie: "Inception", genre: "Science Fiction"},
	{movie: "Inception", genre: "Thriller"},
	{movie: "Interstellar", genre: "Science Fiction"},
	{movie: "The Dark Knight", genre: "Action"},
	{movie: "The Dark Knight", genre: "Crime"},
	{movie: "The Dark Knight Rises", genre: "Action"},
	{movie: "Dunkirk", genre: "War"},
	{movie: "The Revenant", genre: "Adventure"}
] AS item
MATCH (m:Movie {title: item.movie})
MERGE (g:Genre {name: item.genre})
MERGE (m)-[:IN_GENRE]->(g);
```

### 6. Recommend movies similar to Inception

This read-only query scores candidates based on shared actors (3 points each),
shared directors (2 points each), and shared genres (1 point each). It excludes
the favorite movie itself and returns candidates with at least one shared feature.

```cypher
MATCH (favorite:Movie {title: "Inception"})
MATCH (rec:Movie)
WHERE rec <> favorite

OPTIONAL MATCH
		(favorite)<-[:ACTED_IN]-(actor:Person)-[:ACTED_IN]->(rec)
WITH favorite, rec,
		 count(DISTINCT actor) AS sharedActors

OPTIONAL MATCH
		(favorite)<-[:DIRECTED]-(director:Person)-[:DIRECTED]->(rec)
WITH favorite, rec, sharedActors,
		 count(DISTINCT director) AS sharedDirectors

OPTIONAL MATCH
		(favorite)-[:IN_GENRE]->(genre:Genre)<-[:IN_GENRE]-(rec)
WITH rec, sharedActors, sharedDirectors,
		 count(DISTINCT genre) AS sharedGenres

WITH rec, sharedActors, sharedDirectors, sharedGenres,
		 sharedActors * 3 + sharedDirectors * 2 + sharedGenres AS score
WHERE score > 0

RETURN rec.title AS recommendation,
			 sharedActors,
			 sharedDirectors,
			 sharedGenres,
			 score
ORDER BY score DESC, recommendation;
```
