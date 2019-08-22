# lec-graph

A Neo4j graph using the League of Legends European Championship data.

The data was scraped from [Leaguepedia](https://lol.gamepedia.com/League_of_Legends_Esports_Wiki).

## Database model

```
MATCH (m:Match)
WITH m
LIMIT 1
RETURN m, (m)--(:Team), (m)--(:Caster)
```

![Diagem](/img/database-model.png)

## Import the data

Copy the following files into the `import` folder of your Neo4j installation.

- teams.csv
- matches.csv
- casters.csv

Then run the following queries to import the data.

```
LOAD CSV WITH HEADERS FROM "file:///teams.csv" AS row
MERGE (t:Team { nameShort: row.nameShort, nameFull: row.nameFull })
```

```
LOAD CSV WITH HEADERS FROM "file:///matches.csv" AS row
MATCH (t1:Team { nameShort: row.team1 }),
      (t2:Team { nameShort: row.team2 }),
      (winner:Team { nameShort: row.winner })
MERGE (m:Match { date: row.date, team1: row.team1, team2: row.team2 })
MERGE (t1)-[:PLAYED]->(m)<-[:PLAYED]-(t2)
MERGE (winner)-[:WON]->(m)
```

```
LOAD CSV WITH HEADERS FROM "file:///casters.csv" AS row
MATCH (m:Match { team1: row.team1, team2: row.team2 })
MERGE (c:Caster { name: row.casterName, type: row.casterType })
MERGE (c)-[:CASTED]->(m)
```

## Queries

### Number of matches casted per Caster

```
MATCH (m:Match)
WITH COUNT(m) AS numberOfMatchesTotal
MATCH (c:Caster)-[:CASTED]-(m:Match)
RETURN c.name AS casterName, 
       COUNT(m) AS numberOfMatchesCasted, 
       COUNT(m) * 100.0 / numberOfMatchesTotal AS percentageOfMatches
ORDER BY numberOfMatchesCasted DESC
```

### Teams casted by Medic & Vedius

```
MATCH (:Caster {name: "Medic"})-[:CASTED]->(m:Match)<-[:CASTED]-(:Caster {name: "Vedius"})
WITH m
MATCH (t:Team)-[:PLAYED]->(m)
RETURN t.nameFull AS teamName, 
       COUNT(*) AS numberOfTimesCasted
ORDER BY numberOfTimesCasted DESC
```

### Matches which had a tri-cast

```
MATCH (t:Team)-[:PLAYED]->(m:Match)<-[:CASTED]-(c:Caster)
WITH m, 
     COLLECT(DISTINCT t.nameFull) AS matchup, 
     COLLECT(DISTINCT c.name) AS cast
WHERE size(cast) = 3
RETURN m.date AS date, 
       matchup[0] + " VS " + matchup[1] AS Match,
       cast AS Cast
ORDER by date ASC
```

### Casters who never casted together

```
MATCH (c1:Caster),
      (c2:Caster)
WHERE id(c1) < id(c2) 
  AND NOT( (c1)-[:CASTED]->(:Match)<-[:CASTED]-(c2) )
RETURN c1.name,
       c2.name
```
