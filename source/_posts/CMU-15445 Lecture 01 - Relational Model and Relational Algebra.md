---
layout: post
date: 2022-01-23
title: "CMU 15-445 Lecture 01: Relational Model and Relational Algebra"
categories:
tags: 
mathjax: true
---

Database: collection of data that relates to gather and tries to model some aspect of the world.

## Flat Files

### Strawman

we store our database as comma-separated value (CSV) files that we manage in our own code. 

* Use separated files for each entity.
* Application has to parse the files each time they want to read/update records.

<!-- more -->

For example: a music database

```
Artist.CSV (name, year, country)
"Wu Tang Clan", 1992, "USA"
"Ice Cube", 1989, "USA"
...

Album.CSV (namme, artist, year)
"Enter the Wu Tang", "Wu Tang Clan", 1993
"AmeriKKKa's Most Wanted", "Ice Cube", 1990
...
```

If we want to get the year that "Ice Cube" went solo, we can write the python code:

```python
for line in Artist.CSV:
	record = parse(line)
    if "Ice Cube" == record[0]
    	print int(record[1])
```

### Drawback: Data Integrity

* How do we ensure that the artist is the same for each album entry? e.g. Ice Cube modifies its name, how do we ensure that we modify all the "Ice Cube" in the database?

* What if somebody the album year with an invalid string?

    There's no protection mechanism in our database now, all the files are regular files that anyone has access to.

* How do we store if there are multiple artists on an album?

* ...

### Drawback: Implementation

* How do you find a particular record? 

    Linear searching will be too slow if there are huge amounts of entries in the database.

* What if we now want to create a new application that uses the same database?

* What if two threads try to write to the same file at the same time?

* ...

### Drawback: Durability

* What if the machine crashes while our program is updating a record?
* What if we want to replicate the database on multiple machines for high avalability?
* ...

## Database Management System

A **DBMS** is a specialized software that allows applications to store and analyze information in a database. A general purpose DBMS is designed to allow the definition, creation, querying, update, and administration of database.

## Relational Model

Background: in 1960s, people rewrote DBMS every time the physical layer was changed.

3 key points about relational model:

* store database in simple data structures (relation)
* access data through high-level language (e.g. SQL)
* physical storage left up to implementation i.e. the specific implementation of storage(disk, memory etc.) is transparent to the applications, which is good for portability.

The relational data model defines three concepts:

* Structure: the definition of relations and their contents
* Integrity: ensure that database's contents satisfy constraints
* Manipulation: how to access and modify a database's contents

### Concepts

A **Relation** is an unordered set that contains the relationship of attributes that represent entities. (In the picture, a table is a relation)

A **tuple** is a set of attribute values (domains) in the relation. (In the picture, a "row" in the table is a tuple with several attribute values (name, year, country etc.) ) Conventionally, attribute values have to be atomic/scalar i.e. arrays, lists... cannot be stored as an attribute value.

<img src="https://kristoff-starling.github.io/files/CMU-15445-01-01.png" width="80%"/>

A **primary key** is an attribute that can uniquely identify a single tuple. For example, "year" is not a primary key since it's not unique in the table; "Wu Tang Clan" is also not a primary key because it's allowed that in the future an artist with the same name will be created. The id is the primary key. Some DBMSs automatically create an internal primary key if you don't define one.

A **foreign key** specifies that an attribute from one relation maps to a tuple in another relation. For example, the `album_id` attribute 11 in `ArtistAlbum` table is a foreign key since it maps to "Enter the Wu Tang".

(Usually) attribute values should be scalar so we cannot store multiple artists into one album. But by adding an additional relation `ArtistAlbum` shown above, we can achieve that. In addition, the DBMS can check whether the given `artist_id` and `album_id` are valid.

### Data Manipulation Languages (DML)

DML describes how to store and retrieve data from a database. There are mainly two types:

* Procedural: the query specifies the (high-level) strategy the DBMS should use to find results.

    Relational Algebra

* Non-Procedural: the query specifies only what data is wanted and not how to find it.

    Relational Calculus

## Relational Algebra

### Basic Operators

#### Select

* Syntax: $\sigma_{predicate}(R)$

* Description: choose a subset of the tuples from a relation that satisfies a selection predicate. Here the predicate serves as a filter to retain only tuples that fulfill its qualifying requirement.

* Example:

    <img src="https://kristoff-starling.github.io/files/CMU-15445-01-02.png" width="50%" />

    In SQL, it's written as

    ```sql
    SELECT * FROM R
    	WHERE a_id = 'a2' AND b_id > 102;
    ```

#### Projection

* Syntax: $\Pi_{A_1,A_2,\cdots,A_n}(R)$

* Description: generate a relation with tuples that contains only the specified attributes. We can rearrange attributes' ordering and manipulate the values.

* Example (We can combine operators to achieve abundant functionality):

    <img src="https://kristoff-starling.github.io/files/CMU-15445-01-03.png" width="50%" />

    In SQL, it's written as

    ```sql
    SELECT b_id - 100, a_id
    	FROM R WHERE a_id = 'a2';
    ```

#### Union

* Syntax: $(R \cup S)$
* Description: the same as set algebra. Notice that $R$ and $S$ should have the exact same attributes.
* SQL example:

    ```sql
    (SELECT * FROM R)
    	UNION ALL
    (SELECT * FROM S);
    ```

#### Intersection

* Syntax: $(R\cap S)$

* Description: the same as set algebra. Notice that $R$ and $S$ should have the exact same attributes.

* SQL example:

    ```sql
    (SELECT * FROM R)
    	INTERSECT
    (SELECT * FROM S);
    ```
#### Difference

* Syntax: $(R - S)$

* Description: the same as set algebra. Notice that $R$ and $S$ should have the exact same attributes.

* SQL example:

    ```sql
    (SELECT * FROM R)
    	EXCEPT
    (SELECT * FROM S);
    ```

#### Product

* Syntax: $(R\times S)$
* Description: the same as set algebra (Cartesian product)

* SQL example:

    ```sql
    SELECT * FROM R CROSS JOIN S;
    ```

#### Join

* Syntax: $(R\Join S)$

* Description: Join takes in two relations and outputs a relation that contains all the tuples that are a combination of two tuples where for each attribute that the two relations share, the values for that attribute of both tuples is the same.

    The difference between "Join" and "Intersection" is that, Join doesn't require that the two relations have the exact same attributes, it only requires that their common attributes should be the same.

* SQL example:

    ```sql
    SELECT * FROM R NATURAL CROSS S;
    ```

### Observation

Relational algebra is a kind of procedural language because it defines the high-level steps of how to compute a query. For example:

> We want to get all the tuples in relation R and S whose `a_id` equals 102.

$\sigma_{a\_id=102}(R\Join S)$ and $R\Join(\sigma_{a\_id=102}(S))$ can both achieve the goal, but their efficiency may vary much. Suppose $R$ and $S$ both have $a$ tuples, but there's only one tuple in $S$ satisfying $a\_id=102$. 

* In the first expression, $\Join$ may take $a^2$ complexity, and $\sigma$ finds only one tuple. - $O(a^2)$
* In the second expression, $\sigma$ takes $a$ complexity and finds only one valid tuple, and then $\Join$ also only takes $a$ complexity. - $O(a)$

If $a$ is huge enough, the second operations is much faster than the first one.

A better approach is to say the result you want, and let the DBMS decide the steps it wants to compute a query according to the data size. SQL will do exactly this, and it is the de facto standard for writing queries.

Compare the SQL query with our strawman's python query:

```python
for line in Artist.CSV:
	record = parse(line)
    if "Ice Cube" == record[0]
    	print int(record[1])
```

```sql
SELECT year FROM artists
	WHERE name = "Ice Cube"
```

SQL's robustness is much stronger. It doesn't depend on the implementation of database, so if the low-level organization of the database is changed (e.g. using a tree instead of a list to store data), the application layer doesn't need to change (while the python code becomes completely useless.)

