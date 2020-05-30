---
title: The Ultimate Guide to Learning Neo4j with Netflix's Dark Series
description: Learn Graph Databases with Neo4j and Netflix's Dark Series
author: James Ingold
published: true
---

Netflix's Dark is the best TV series ever created. Hands down. Dark is a science fiction mind-bender and time twister where children in the town of Winden go missing. Folks in the town start asking questions and before you know it your head is spinning through multiple different universes as time unravels. Graph databases were built to handle highly-connected and complicated data. I can't think of better adverbs for Dark. Time and everything else is woven together, highly related and complex. Let's learn abut graph databases and explore Netflix's Dark TV Series as we pick up some Neo4j knowledge.

This post is written just before Season 3 comes out, so we'll be making some predictions in this post.

**SPOILER ALERT - THERE WILL BE SPOILERS**

#### Goals

- To learn about Graph Databases using Neo4JS
- To gain insights into the great Dark series

#### Neo4j (Better Data, Betters Decisions) Crash Course

Neo4j is a flavor of graph database. These types of databases are relational but they're not a traditional relational database. In fact, traditional relational databases are not great at handling highly related data. We'll get more into that later in this post.

Neo4j was originally built on top of MySQL, but it was redesigned from the ground up to focus specifically on optimizing graphs. It features ACID  
(Atomicity, Consistency, Isolation, Durability) transactions:

- Atomicity => All or nothing, if the transaction fails, it's rolled back.
- Consistency => this is debatable as Neo4j is schemaless but we'll give it to them as the schema is in a consistent state.
- Isolation => This requires that multiple transactions which are executed in parallel on the same database instance do not impact each other.
- Durability => Persisted, data in Neo4j is written to disk.

#### Nodes, Labels, Properties, Relationships

A graph database is a collection of nodes which can be labeled, connected by relationships, both of which can have properties. A node is an object or an entity. In our Dark data, we'll have nodes that have labels of Character, Family, Year and Sic Mundus. For example, Jonas Kahnwald is a character in Dark and a node in our graph. He is related to another node Hannah Kahnwald. This relationship is that Hannah is his parent. We have a PARENT_OF relationship. Jonas could also have a CHILD_OF relationship pointing back to Hannah. There can be multiple relationships between nodes. These relationships are what connect nodes in the graph. The id of a node is an auto incrementing integer but it can also be set manually. Properties are key value pairs where key is a string and value is a primitive. Properties can be indexed, if the property for the index is not there, the node simply is not indexed. Properties can also have constraints.

Graph databases are schemaless, meaning they are highly flexible. You can easily add properties to new nodes. Relational databases have nullable columns, but for graph databases, you just don't add the properties. This gives us some nice document database (MongoDB) features with highly relational data.

#### Why not just use a traditional relational database?

Relational databases are actually quite anti-relational. There is a limit to the number of join operations that a system can effectively perform, before the "join bombs" go off and the system becomes very unresponsive. Anyone who has spent time with an RDMS knows this intuitively but it's not clear until something better comes along to offer you the red pill. (Neo4j really brings out the Matrix references). Instead of tables like in a relational database, we have nodes. Instead of using joins, we have pattern matching. Joins decrease performance exponentially. Joins are expensive. Relations are cheap. In short, relational databases weren't actually built for deeply related data.

In [Neo4j in Action](https://amzn.to/2LA2UcR) they compare Neo4j against MySQL for a social network. They show that when you have 110,000 records and need to go 3 levels deep, Neo4j takes less than a second while MySQL takes 30,000 seconds. At a certain point, say five levels deep, the relational database is not even usable.

#### When to Use

- Complex queries: when the types of questions that you want to ask of your data are composed of a number of complex join-style operations. Basically, when you have more than 3 joins, prefer pattern matching queries.
- Pathfinding queries: where you will be looking to find out how different data elements are related to each other.

Performance: as soon as you grab a starting node, the database will only explore the things that relate to it and will be completely oblivious to anything that is not connected to the starting node. This makes querying by patterns efficient.

#### When to **not** use a Graph Database:

When the complexity is low. Graph databases are meant for highly related and complex data. If you don't have that, use a traditional relational database or NoSQL solution. You need to envision joining multiple tables together in your project. If you're going to go 3 or more levels deep, take a look at graph databases.

#### Query Language

Cypher is the declarative, pattern-matching query language for Neo4j. It makes graph database management systems understandable and workable for any database user. We'll learn some more about Cypher later in this tutorial.

### Getting Started

Now that we're acquainted, let's get up and running by starting our own instance of Neo4j. We'll grab the Docker image, enabled with apoc (Awesome Procedures Of Cypher) - fun fact - Apoc kills Cypher in the Matrix. Apoc isn't going to rid our use of Cypher but it is the home to [lots of cools functions and graph algorithms](https://neo4j.com/docs/labs/apoc/current/overview/){:target="\_blank"}. Like most databases, Neo4j stores it's data on the file system (Durability), so we will map some folders to our local system that Neo4j in Docker can use.

```bash
docker pull neo4j
```

```bash
docker run \
    --name neo4j \
    -p7474:7474 -p7687:7687 \
    -d \
    -v $HOME/neo4j/data:/data \
    -v $HOME/neo4j/logs:/logs \
    -v $HOME/neo4j/import:/var/lib/neo4j/import \
    -v $HOME/neo4j/plugins:/plugins \
    --env NEO4J_AUTH=neo4j/securepassword \
    --env NEO4JLABS_PLUGINS='["apoc"]' \
    --env apoc.import.file.enabled=true \
    neo4j:latest
```

Let's check the logs and make sure we're up and running

```bash
docker container logs testneo4j -f
```

![Docker Neo4j Logs](https://dev-to-uploads.s3.amazonaws.com/i/8yvkesx4jzq6b37au7ue.png)

Ports we exposed:  
7474: Neo4J Browser (View our data in a local web browser) <br />
7687: Bolt port (how we communicate with the database)

We mapped some folders: data, logs, plugins to our home/neo4j directory. We created a neo4j user with "securepassword" as the password.

Open up the Neo4j Browser at http://localhost:7474/browser/ and make sure it loads.

#### Setting up our Dev Environment

Now that we've got a Neo4j instance up and running, let's get a couple things in order for our development environment. Here are a few tools to use when working with Neo4j and the Cypher query language. These are handy for writing Cypher but if you want to stay just in the Neo4j Browser, that will work perfectly fine as well.

VSCode/Codium:

This is an excellent Cypher extension for VSCode editors: [agatlin/vscode-cypher-query-language-tools](https://github.com/agatlin/vscode-cypher-query-language-tools){:target="\_blank"}.

Vim: [Syntax highlighting](https://github.com/neo4j-contrib/cypher-vim-syntax){:target="\_blank"}

#### Data

I created my own Dark data set which was heavily influenced by this great [reddit post](https://i.redd.it/3nypv8br5f301.jpg){:target="\_blank"} from season one. This [Gitub repository](https://github.com/james-ingold/netflix-dark-neo4js){:target="\_blank"} will contain characters.json which you will need to move to \$HOME/neo4j/import to be able to use the corresponding cypher import script below.

```bash
cp characters.json $HOME/neo4j/import
```

Open up characters.json and we'll take a look at the data. There are several relations defined in there, this is a series about a small town in Germany so mainly we're talking about families. People also tend to die of non-nature causes.

siblings <br />
parents <br />
parentOf <br />
married <br />
killedBy <br />
sicMundus

Relations can also contain properties, so killedBy could have date.

The characters.json file defines character objects and the relationships above. Here is a json representation of our main character Jonas Kahnwald.

```json
{
  "name": "Jonas Kahnwald",
  "lastname": "Kahnwald",
  "alias": ["Stranger", "Adam"],
  "years": [1921, 1953, 1986, 2019, 2052, 2085],
  "parents": ["Hannah Kanhwald", "Mikkel Nielsen"],
  "sicMundus": true
}
```

### The Plan => Cypher Language and Importing Data

Now that we've had a good look at the data, let's import it. Take a look at the [import cypher file](https://github.com/james-ingold/netflix-dark-neo4js/blob/master/import/dark-import.cypher){:target="\_blank"}. This is the code we will run to import characters.json into the database. Once you've read the below comments, run the file in the Neo4j Browser.

We'll start out by creating some constraints for our character and year node. We're indexing families and characters on name. While, we won't run into performance issues with this small of a data set, it is good practice to get into the habit.

```cypher
CREATE constraint on (c:Character) assert c.name is unique;
CREATE constraint on (y:Year) assert y.id is unique;
CREATE constraint on (s:SicMundus) assert s.name is unique
CREATE index on :Character(name);
CREATE index on :Family(name);
```

Next, we'll manually create some nodes for years and family. We could have done this using the json but I just wanted to get up and running:

```cypher
CREATE (`1921`:`Year` {id: 1921}), (`1953`:`Year` {id: 1953}),
...
```

Enter apoc. Now we'll use an Awesome Procedure on Cypher function to import our character.json file. We reference it by the word _value_, then we'll UNWIND the array of characters, basically iterating through each character object in the json. We'll use another apoc function to clean up any empty or nulls. From there we create a new Character node with a name and then attach the rest of the properties.

```cypher
CALL apoc.load.json("file:///characters.json") YIELD value
UNWIND value.characters AS character
WITH apoc.map.clean(character, [],['',[''],[],null]) as data
MERGE (c:Character {name:data.name})
SET
c += apoc.map.clean(data, [],['',[''],[],null])
```

Now onto setting up relationships. Mapping our array relationships is fairly straight forward. We'll loop through each relationship property and add the appropriate relationship. We're using MERGE which will create a node or use an existing. Remember, we have a unique constraint on character name. Notice how the relationships can go either way.

```cypher
FOREACH (name in data.married | MERGE (m:Character {name:name}) MERGE (m)-[:MARRIED]->(c))
FOREACH (name in data.parentOf | MERGE (m:Character {name:name}) MERGE (c)-[:PARENT_OF]<-(m))
FOREACH (name in data.siblings | MERGE (m:Character {name:name}) MERGE (m)-[:SIBLING_OF]->(c))
FOREACH (name in data.killedBy | MERGE (m:Character {name:name}) MERGE (c)-[:KILLED_BY]<-(m))
FOREACH (id in data.years | MERGE (y:Year {id:id}) MERGE (c)-[:WHEN]->(y))
```

We have a couple properties to relate to nodes, Sic Mundus and Family which are just primitives (boolean and string). We use case to test if the property is null and then empty it out before once again using MERGE to create our nodes and relationships.

```cypher
FOREACH (name in case data.lastname when null then [] else [data.lastname] end | MERGE (f:Family {name:name}) MERGE (c)-[:FAMILY]->(f))
FOREACH (name in case data.sicMundus when null then [] else [data.sicMundus] end | MERGE (s:SicMundus) MERGE (c)-[:SIC_MUNDUS_MEMBER]->(s))

```

Perfect, now let's view entire graph in the Neo4J Browser

```cypher
MATCH (n) RETURN n
```

![Neo4j Dark Graph](https://dev-to-uploads.s3.amazonaws.com/i/42xptiszzkbryjyhtp43.png)

#### Basic Cypher Quick Hits

Now that we've got a populated Neo4j database and an introduction to Cypher, let's dive a little deeper on this steak loving declarative querying language.

CREATE: This creates nodes and relationships with their properties.

```cypher
CREATE (`Jonas Kahnwald`:`Character`{ name: 'Jonas Kahnwald' })-[:CHILD_OF]->(`Hannah Kahnwald`:`Character`{ name: 'Hannah Kahnwald' })
```

We've created two nodes Jonas and Hannah and added a relationship of CHILD_OF. Relationships can go both ways or we could reverse it like this:

```cypher
CREATE (`Jonas Kahnwald`:`Character`{ name: 'Jonas Kahnwald' })<-[:PARENT_OF]-(`Hannah Kahnwald`:`Character` {name: 'Hannah Kahnwald'})
```

MATCH: basically like a SQL select, it describes a pattern that should _match_ in the database.

WHERE: pretty much like SQL, filters the results.

A difference between SQL and Cypher is that in the latter, you return the data that you want declaratively. As we saw above, this Cypher will return the entire database. Match all nodes and then return them.

```cypher
MATCH (n) RETURN n
```

We can query for relationships as well, this will look at all the Years when Magnus Nielsen has been present. We are just returning the nodes here but we could return properties such as c.name, y.id, etc.

```cypher
MATCH(c:Character)-[r:WHEN]->(y:Year) where c.name = 'Magnus Nielsen' return c, y
```

![Magnus all whens](https://dev-to-uploads.s3.amazonaws.com/i/pt8hixq73wcopk14ophu.png)

We could make a character, not a character by removing a label:

```cypher
MATCH (c:Character)
WHERE c.name = 'Magnus Nielsen'
REMOVE c:Character
RETURN c ;
```

Remove a node:

```cypher
MATCH (c.Character, {name: 'Magnus Nielsen'})
DELETE c
```

Use detach delete to remove relations and then delete node:

```cypher
 MATCH (c.Character {name: 'Magnus Nielsen'})
 DETACH DELETE c;
```

WITH: This is a handy keyword which passes results from one query part to the next. WITH is like RETURN; it allows you to separate the query in parts. You'll see this in action in the next section.

MERGE: As mentioned above, this is like CREATE but it won't fail if the node or relationship already exists.  
Contrived example of WITH and MERGE

```cypher
MATCH (h: {name: "Hannah Kahnwald"}) as hannah
WITH MATCH(j:User {name: "Jonas Kahnwald"}) jonas
MERGE (hannah)-[:PARENT_OF]->(jonas)
```

### Questions and Findings

Now we have more than enough Neo4j knowledge to be dangerous. Let's get to the fun part, finding some patterns in our dataset! When creating a graph database, it's good to know what kind of questions you will want to ask the data. Here is a list of questions and queries:

Who is the most mysterious (least amount of relations)?

```cypher
MATCH(c:Character)-[r]-(x) RETURN c.name, COUNT(*) as relations
ORDER BY relations limit 5
```

Bartosz Tiedeman and Clausen each come in with 2 nodes. Yasin Friese only has one, however Clausen and Bartosz are likely key characters who we will find out more about in Season 3.

Are there any characters not related to a family?

```cypher
MATCH (c:Character) WHERE NOT (c)-[:FAMILY]->() RETURN c
```

Nope, every character has a family.  
A better question,  
Which family has the most members?

```cypher
MATCH(c:Character)-[r:FAMILY]-(f:Family) RETURN count(c) as cnt, f.name order by cnt desc
```

Despite Jonas being the main character, the Nielsen's have the most known family members at 9. While the Kahnwalds sit at 4, not including Mikkel.  
The Doppler's come in at 7.

Which "family" has only one member?

```cypher
MATCH(c:Character)-[r:FAMILY]-(f:Family)
WITH count(c) as cnt, f
WHERE cnt = 1
RETURN f
```

![One Member Family Graph](https://dev-to-uploads.s3.amazonaws.com/i/8ima1f6xcyus191uj800.png)

What are the most active time segments?

```cypher
MATCH(c:Character)-[r:WHEN]->(y:Year) RETURN COUNT(c.name) as cname, y.id
```

2019 is obviously the most when'd timeverse as that's where Dark starts. As the timeverses edge out, we know less characters who have traveled there.

Who has gone the farthest in time?

```cypher
MATCH(c:Character)-[r:WHEN]->(y:Year) RETURN c.name, count(r) as whens ORDER BY whens desc
```

Jonas is the only confirmed character to have traveled to 2085. We will probably learn more about that year in Season 3. He has also seen the most timeverses.

Which characters are in Sic Mundus?

```cypher
MATCH(c:Character)-[:SIC_MUNDUS_MEMBER]->(s:SicMundus) RETURN c
```

![SicMundus Members](https://dev-to-uploads.s3.amazonaws.com/i/1lkzieyuxj9x15v8n5jl.png)

Now for our most ambitious query of this post, who are the likely other Sic Mundus members?

<img src="https://dev-to-uploads.s3.amazonaws.com/i/d9jpaqnprvcygqbjbb0r.jpeg" height="400" alt="SicMundus" style="display:block;margin-left:auto;margin-right:auto">

This type of query can give you a hint to what Neo4j can do in terms of predictions or recommendations out of the box. Notice how we can specify which relationships we're looking for with -> and -

```cypher
MATCH(c:Character)-[:SIC_MUNDUS_MEMBER]->(s:SicMundus)
WITH c
MATCH(c)-[r]-(x:Character)
WHERE not (x)-[:SIC_MUNDUS_MEMBER]->() AND not (x)-[:KILLED_BY]->()
RETURN x.name, count(r) as cnt
ORDER BY cnt desc
```

"Martha Nielsen" 2 <br />
"Elisabeth Doppler" 2 <br />
"Charlotte Doppler" 2 <br />

Martha, Elisabeth, and Charlotte all seem likely Sic Mundus candidates. This query could be much higher quality if we had even more data points to look at.

Which characters have only been to one timeverse?

```cypher
MATCH(c:Character)-[r:WHEN]->(y:Year) WITH c.name as name, count(r) as whens WHERE whens = 1 RETURN name
```

"Greta Doppler"<br/>
"Doris Tiedeman" <br />
"Daniel Kahnwald" <br />
"Mads Nielsen" <br />
"Torben Woller" <br />
"Erik Obendorf" <br />
"Bartosz Tiedeman" <br />
"Ulla Obendorf" <br />
"Peter Doppler" <br />
"Yasin Friese" <br />
"Jurgen Obendorf" <br />
"Clausen" <br />

Most characters don't skip time frames, if you've been to 1986, you've likely been to 1953 or 2019. However, some characters have probably warped around the 33 years time frames, let's find them.
There are a few new functions to explain:  
collect will turn a group of nodes or properties into an array.  
Size is like a .length or .count - it gets the size of the array.  
Reduce lets you do some iteration over a collection.

```cypher
MATCH(c:Character)-[r:WHEN]->(y:Year)
WITH c as character, y
ORDER by y.id
WITH character, reverse(collect(y.id)) as years
WITH character, years, range(0, size(years)-1) AS is
WITH character, years, is, years + (last(years) - 33) as isa
WITH character, years, reduce(total=0, n in is | (total + toInteger((isa[n] - isa[n+1]) / 33 - 1))) as sum
WHERE sum > 0
RETURN character.name, years
```

"Franziska Doppler" [2019, 1921] <br />
"Magnus Nielsen" [2052, 2019, 1921]

I think it's a good bet that we see a lot more of Magnus and Franziska in Season 3 as most characters don't skip around the time frames.

Which characters have never been to the past?

```cypher
MATCH(c:Character)-[r:WHEN]->(y:Year)
WITH c as character, MIN(y.id) as my
WHERE my >= 2019
RETURN character.name
```

Bartosz is questionable - there is a theory that Noah killed him in back in 1921. Bartosz as noted above is one of the more mysterious characters and there are rumors that he is also a Sic Mundus member. Will these characters travel further back in season 3?  
"Torben Woller" <br />
"Erik Obendorf" <br />
"Bartosz Tiedeman" <br />
"Ulla Obendorf" <br />
"Peter Doppler" <br />
"Martha Nielsen" <br />
"Elisabeth Doppler" <br />
"Yasin Friese" <br />
"Jurgen Obendorf" <br />
"Clausen"

Which characters are the most deadly?  
<> means not equals or does not match.

```cypher
MATCH(c:Character)-[r:KILLED_BY]->(k:Character)
WHERE c.name <> k.name
RETURN c.name, k.name
```

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/souiovdvmy3cr98nqlxi.png)

Noah has killed Claudia Tiedeman, Mads Nielsen. You could probably add Yasin to this list as well and possibly Bartosz.  
Agnes Nielsen killed Noah.  
The Nielsen family is the most deadly. What does that mean for Season 3?

What percentage of deaths in Dark have been suicides?

```cypher
MATCH(c:Character)-[r:KILLED_BY]->(k:Character)
WHERE c.name <> k.name
WITH count(c) as cnt
MATCH(s:Character)-[r:KILLED_BY]->(sk:Character)
WHERE s.name = sk.name
RETURN toFloat(count(s)) / (count(s) + cnt)
```

40%

### Conclusion

Hopefully this post has helped you get up to speed with Neo4j using data from Netflix's Dark Series. If you noticed while running the queries, all of them take under 10ms. Crazy fast. There are definitely ways to improve the Dark dataset. For instance, we could have a relationship of APPEARS_IN with each season being a node. There could also be an INTERACTS_WITH relationship when characters interact with each other and a count on that relationship. These queries may not be the most efficient to express our questions.

What do you think? Have better ways to create the data model? Got some cool queries to run? Want to add to the Dark dataset? Checkout the [Github repo](https://github.com/james-ingold/netflix-dark-neo4js){:target="\_blank"}.

#### Further Reading / References / Tools

[Learning Neo4J v3.x](https://amzn.to/3eCSK8b){:target="\_blank"}

[Running Neo4j in Docker](https://neo4j.com/developer/docker-run-neo4j/){:target="\_blank"}

[Docker Neo4JS](https://github.com/neo4j/docker-neo4j/){:target="\_blank"}

[Neo4J CheatSheet](https://gist.github.com/DaniSancas/1d5265fc159a95ff457b940fc5046887){:target="\_blank"}

[Dark Wikipedia](<https://en.wikipedia.org/wiki/Dark_(TV_series)>){:target="\_blank"}

[Dark Character Map](https://www.bustle.com/p/this-dark-character-map-will-help-you-make-sense-of-the-shows-tangled-web-of-characters-timelines-18160529){:target="\_blank"}

[Arrows - Open Source Library for Drawing Graphs](https://github.com/apcj/arrows){:target="\_blank"}
