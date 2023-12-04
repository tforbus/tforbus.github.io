---
title: "Schemaless Postgres"
date: 2023-12-03T18:41:44-08:00
tags: ["database", "nosql", "postgres"]
---

I stumbled across a link on HackerNews that [Reddit's database has two tables](https://news.ycombinator.com/item?id=32407873).
Inside was a blog post from 2012 that referenced a presentation by Steve Huffman from 2010. The interesting thing is how Reddit effectively had no schema for their data, even though they were (at least at the time) using Postgres for storage. The gist is there are two tables. One table has the metadata about every *thing* that Reddit has: users, subreddits, comments, etc.  The second table contains all the data about a *thing*, but each attribute of that data is spread across a new row. If you read the HackerNews link, you'll find that the pros and cons of this approach are a religious debate. After really only using Key-Value and Document databases, I couldn't shake the desire to try this out.

And just to clarify, I have so little experience with Postgres we can round it down to zero.

## Schemaless users
{{<highlight sql>}}
-- Using INT as an identifier here so the content is easier to follow.
CREATE TABLE IF NOT EXISTS users_noschema (
    id INT NOT NULL,
    attr VARCHAR NOT NULL,
    attr_data JSONB,
    CONSTRAINT unique_id_attr PRIMARY KEY (id, attr)
);
{{</highlight>}}

With this psql, we create a table with only 3 columns, with id+attr making up a primary key. We've successfully created a key-value store. The *attr_data* column can store data of any kind, and we can cast it to whatever we want later. If we want to put something in our table, we can execute a query to insert arbitrary data for arbitrary attributes on a user.

{{<highlight sql>}}
-- Creating a new user with one attribute email
INSERT INTO users_noschema(id, attr, attr_data)
VALUES (0, 'email', '"bob@example.com"');
{{</highlight>}}

But writing a user like this, one attribute at a time seems like a drag. If the user was just one row, we could perform one write and be done. To write multiple values at once, we just need to extend the first insert statement a bit.

{{<highlight sql>}}
-- Create a user with multiple attributes
INSERT INTO users_noschema(id, attr, attr_data)
VALUES
    (1, 'email', '"alice@example.com"'),
    (1, 'favoriteNumber', '42');
{{</highlight>}}

So now we've inserted an Alice into our database and given her two properties: email, and favorite nummber. One thing we can see from here is that if our user has *N* properties at creation time, then we'll perform *N* writes to the database, and likewise if we want to read an entire user, we'll read *N* rows.

To read Alice and all her properties, we could do a simple query selecting on her ID.

{{<highlight sql>}}
SELECT * FROM users_noschema
WHERE id='1';
{{</highlight>}}

This will return a result of *N* rows. But because we've got a JSONB column, we could return 1 row!

{{<highlight sql>}}
SELECT id, jsonb_object_agg(attr, attr_data) as user_data
FROM users_noschema
WHERE id='1'
GROUP BY id;
{{</highlight>}}

And this will return a result with 2 columns: id, and user_data. The user_data is a JSONB object where our attributes are keys, and the values are the corresponding attr_data entries. It would look something like the blob below.

| id (integer) | user_data (jsonb) |
|--------------|-------------------|
| 1 | {{<highlight json>}}{"emailAddress": "alice@example.com", "favoriteNumber": 42}{{</highlight>}}|

Pretty cool! We can extend this pattern to query for a page of users, all users, or a projection of a subset of user properties. To get a projection that only has the user email addresses, we can use a query like this:

{{<highlight sql>}}
SELECT id, jsonb_object_agg(attr, attr_data)
FILTER (WHERE attr IN ('emailAddress')) as user_data
FROM users_noschema
GROUP BY id;
{{</highlight>}}

Updating a user can be done one attribute at a time, or with multiple updates as we did with insert. If we need to make updates to Alice, it's easy enough.

{{<highlight sql>}}
UPDATE users_noschema
SET attr_data='7'
WHERE id='1' AND attr='favoriteNumber';
{{</highlight>}}

If we need to make multiple updates to Alice, we can use CASE.

{{<highlight sql>}}
UPDATE users_noschema
SET attr_data = 
    CASE
        WHEN attr = 'emailAddress' THEN '"alice.goth@example.com"'
        WHEN attr = 'favoriteNumber' THEN '666'
        ELSE attr_data
    END
WHERE id='1';
{{</highlight>}}

Deleting a user is straightforward.
{{<highlight sql>}}
DELETE FROM users_noschema WHERE id='1';
{{</highlight>}}

So we can perform the bread and butter CRUD operations on our data. That's a good start for a web application. Postgres allows you to reach into jsonb columns so we can even filter on items of interest. But is this key-value version of a table any good? We've gained a lot flexibility with this key-value approach, and we could even put different types of *things* in this single table. That could potentially reduce some operations we need to worry about. It might simplify some maintenance tasks. Then again, it might not. All of this stuff is really *it depends*, but one thing we can quantify is the performance of this approach vs a traditional schema model.

## Traditional user model
We've created a schemaless user model where we really just have 3 fields: ID, email address, and favorite number. We can model that in a way you might normally do in a relational database.

{{<highlight sql>}}
-- Using INT as an identifier here so the content is easier to follow.
CREATE TABLE IF NOT EXISTS users_schema (
    id INT NOT NULL,
    email_address VARCHAR,
    favorite_number INT,
    CONSTRAINT unique_id PRIMARY KEY (id)
);
{{</highlight>}}

I'm going to breeze by CRUD operations here, because it will look sane to anyone who's written a SQL query before.

{{<highlight sql>}}
-- Create Alice
INSERT INTO users_schema(id, email_address, favorite_number)
VALUES (1, 'alice@example.com', 42);

-- Retrieve Alice
SELECT * FROM users_schema WHERE id='1';

-- Update Alice
UPDATE users_schema
SET
	email_address = 'alice.goth@example.com',
	favorite_number = '666'
WHERE id = '1';

-- Delete Alice
DELETE FROM users_schema WHERE id='1';
{{</highlight>}}

## Measuring performance
We can use the EXPLAIN ANALYZE tool to get some details about our queries. 
This tool accepts a statement, executes it, and then provides a query plan. My understanding of the query plan is Postgres is figuring out the best way to execute your query and get your data back. From the output of this, we're looking for the shortest planning and execution times, as well as cost. From the [Postgres docs on EXPLAIN]():
> The most critical part of the display is the estimated statement execution cost, which is the planner's guess at how long it will take to run the statement (measured in cost units that are arbitrary, but conventionally mean disk page fetches). Actually two numbers are shown: the start-up cost before the first row can be returned, and the total cost to return all the rows. For most queries the total cost is what matters...

{{<highlight sql "hl_lines=2">}}
-- Create Alice (explain)
EXPLAIN ANALYZE
INSERT INTO users_schema(id, email_address, favorite_number)
VALUES (1, 'alice@example.com', 42);
{{</highlight>}}

Here's an output I got from running this command on a local Postgres instance.

```
 Insert on users_schema  (cost=0.00..0.01 rows=0 width=0) (actual time=0.173..0.175 rows=0 loops=1)
   ->  Result  (cost=0.00..0.01 rows=1 width=40) (actual time=0.003..0.005 rows=1 loops=1)
 Planning Time: 0.037 ms
 Execution Time: 0.205 ms
(4 rows)
```

We can compare this to inserting with our schemaless design.

{{<highlight sql "hl_lines=2">}}
-- Create a user with multiple attributes
EXPLAIN ANALYZE
INSERT INTO users_noschema(id, attr, attr_data)
VALUES
    (1, 'email', '"alice@example.com"'),
    (1, 'favoriteNumber', '42');
{{</highlight>}}

```
 Insert on users_noschema  (cost=0.00..0.03 rows=0 width=0) (actual time=0.121..0.121 rows=0 loops=1)
   ->  Values Scan on "*VALUES*"  (cost=0.00..0.03 rows=2 width=68) (actual time=0.004..0.006 rows=2 loops=1)
 Planning Time: 0.047 ms
 Execution Time: 0.140 ms
(4 rows)
```

This is just an n=1 result on a toy table, but we can already see a difference in the cost of these queries.

## The experiment
Reddit scaled to [7.5 million users and 270 million page views](http://highscalability.com/blog/2010/5/17/7-lessons-learned-while-building-reddit-to-270-million-page.html) per month using a schemaless design.
We want to measure how a schemaless design performs against a schema design for a database of non-trivial size. We'll seed the database with users, record how long it takes to write each user, and then run the queries for retrieve, update and list some number of times to gain confidence in our numbers. 

The pseudocode for what's happening is as follows:
```
// seed database with one million users
for i := 0; i < 1_000_000; i++ {
    create a user
    record cost, randomly up until 1000 samples recorded.
}

for i := 0; i < 1000; i++ {
    get a user
    record cost

    update a user
    record cost

    get multiple users (list)
    record cost
}
```

The data from the EXPLAIN calls will be sent to another table called performance. We can then query this table to compare results from the schemaless design and schema design. I'm only going to be using total cost, but it is nice to store some additional data for some further investigation. The *operation* in this schema refers to create, retrieve, list, or update, not the operation Postgres is doing under the hood.

{{<highlight sql>}}
CREATE TABLE IF NOT EXISTS performance(
    id SERIAL PRIMARY KEY,
    has_schema BOOLEAN,
    operation VARCHAR,
    startup_cost FLOAT,
    total_cost FLOAT,
    total_time FLOAT,
    execution_time FLOAT
);
{{</highlight>}}

## Results
We can quickly query some basic information about each operation and method with a query like the following:

{{<highlight sql>}}
SELECT
    VARIANCE(total_cost),
    MIN(total_cost),
    MAX(total_cost),
    AVG(total_cost)
FROM performance
WHERE operation='{create|retrieve|update|list}' and has_schema={true|false}
{{</highlight>}}

The results are recorded below.

| Operation | Var | Min | Max | Avg |
| :---      |----: |----: |----: |----: |
| Schemaless create | 0 | 0.040 | 0.040 | 0.040 |
| Schemaless retrieve | 0 | 19.080 | 19.080 | 19.080 |
| Schemaless update | 0 |19.110 | 19.110 | 19.110 |
| Schemaless list | 437152773.482 | 72.820 | 72392.211 | 36232.445 |
| Schema create | 0 | 0.010 | 0.010 | 0.010 |
| Schema retrieve | 0 | 8.440 | 8.440 | 8.440 |
| Schema update | 0 | 8.440 | 8.440 | 8.440 |
| Schema list | 66965118.695 | 28.760 | 28333.770 | 14181.266

Notes:
* There's so much variance with the *list* operations because with each check, the offset is higher.
* I used 3 attributes, rather than 2 in the preceding discussion. Adding 1 more attribute increased the cost of schemaless create by .01, so we can assume how new attributes would cost cost to scale.
* With the exception of list, it looks like there's no need to run these operations multiple times. That makes sense, since we're doing everything based on a primary key.

Overall we can observe that the schemaless approach is more costly than the schema version. It's important to note that JSONB wasn't introduced in Postgres until 2012, 2 years after Huffman's talk. Maybe they used stringy values only in the *attr_data* column and converted to the types they wanted in the queries. That would certainly give some different results. An obvious thing to consider is what if our key-value structure was just an ID and a JSONB column that contained all the user information. Then we'd still have one row per user.

I'd like to explore the second question further: what if our schema was just an (id, jsonb) tuple? Surely the overall costs compared to schemaless would be less.

## Revisiting schemaless

{{<highlight sql>}}
CREATE TABLE IF NOT EXISTS users_noschema2 (
    id INT NOT NULL,
    kvs JSONB,
    CONSTRAINT ns_unique_id PRIMARY KEY (id)
);

-- Creating a user
INSERT INTO users_noschema2(id, kvs)
VALUES(1, '{"email": "alice@example.com", "favoriteNumber": 42 }');
-- Total Cost 0.01

-- Retrieving a user
SELECT * FROM users_noschema2 WHERE id='1';
-- Total Cost 8.17

-- Retrieving a projection of a user
SELECT id, kvs->>'email' FROM users_noschema2 WHERE id='1';
-- Total Cost 8.17

-- Updating a user
UPDATE users_noschema2
SET kvs = '{"email": "alice.goth@example.com", "favoriteNumber": 666}'
-- Total Cost 8.17

-- Updating a particular field
UPDATE users_noschema2
SET kvs = kvs || '{"email": "alice.new@example.com"}'
WHERE id = '1';
-- Total Cost 8.17
{{</highlight>}}

These results already look better than the N-rows-per-attribute version earlier, and they're similar enough to the schema version that either one seems reasonable. But so far we haven't done any queries around filtering the users on some attributes. We've got to seed another schemaless table with one million users first.

Let's filter on the favorite numbers. The datasets don't match up 1-1, but we should be able to figure out a cost comparison.

{{<highlight sql>}}
-- Schema
EXPLAIN (ANALYZE, FORMAT JSON)
SELECT * FROM tmp_schema
WHERE favorite_number > 500;
{{</highlight>}}

{{<highlight json "hl_lines=10-11 15">}}
[
  {
    "Plan": {
      "Node Type": "Seq Scan",
      "Parallel Aware": false,
      "Async Capable": false,
      "Relation Name": "tmp_schema",
      "Alias": "tmp_schema",
      "Startup Cost": 0.00,
      "Total Cost": 87811.54,
      "Plan Rows": 459466,
      "Plan Width": 530,
      "Actual Startup Time": 0.032,
      "Actual Total Time": 607.816,
      "Actual Rows": 498536,
      "Actual Loops": 1,
      "Filter": "(favorite_number > 500)",
      "Rows Removed by Filter": 501464
    },
    "Planning Time": 0.149,
    "Triggers": [
    ],
    "Execution Time": 648.390
  }
]
{{</highlight>}}
I've highlighted the data that seems pertinent here. For the schema table, the difference in the planned rows and actual rows is 39,070.

{{<highlight sql>}}
-- Schemaless, v2
EXPLAIN (ANALYZE, FORMAT JSON)
SELECT * FROM tmp_noschema2 WHERE (kvs->'favoriteNumber')::int > 500;
{{</highlight>}}

{{<highlight json "hl_lines=10-11 15">}}
[
  {
    "Plan": {
      "Node Type": "Seq Scan",
      "Parallel Aware": false,
      "Async Capable": false,
      "Relation Name": "tmp_noschema2",
      "Alias": "tmp_noschema2",
      "Startup Cost": 0.00,
      "Total Cost": 94423.70,
      "Plan Rows": 333328,
      "Plan Width": 599,
      "Actual Startup Time": 0.039,
      "Actual Total Time": 921.486,
      "Actual Rows": 499316,
      "Actual Loops": 1,
      "Filter": "(((kvs -> 'favoriteNumber'::text))::integer > 500)",
      "Rows Removed by Filter": 500684
    },
    "Planning Time": 0.069,
    "Triggers": [
    ],
    "Execution Time": 974.670
  }
]
{{</highlight>}}

Here it looks like storing the data in JSONB affected the query planner. The difference in actual and planned rows is 165,988. Much worse. If we normalize (total cost / actual rows) for each query, the schema version comes in at .176cost/row and the schemaless comes in at .189cost/row. Not much difference in cost, but the planner seems to be worse. Again, my experience with Postgres is effectively 0, but it seems like JSONB columns may perform worse because Postgres can't choose the right plan for the data. I found [this link](https://www.heap.io/blog/when-to-avoid-jsonb-in-a-postgresql-schema) on when to avoid using JSONB which explains the planner a little more. To summarize, Postgres can't keep statistics about the data inside of a JSONB column. The query planner uses those statistics to help it estimate which plan will be the best. In that link, the author was also using JOINs on that blob data, which seemed to have a terrible performance hit.

## Conclusions
Coming from a background of NoSQL and Key-Value databases, I like that Postgres allows arbitrary data in a column. I think for moving fast initially, being able to play fast and loose with your schema is important. When developing an app, and even once you're getting customers, you're still discovering the shape of your data. Having the flexibility to just add data or ignore data without making database migrations is nice. It's probably a good choice if you don't plan on doing any joins or reporting, just like in a normal key-value datastore, but I haven't done this in production yet. I've made it this far in life with just DynamoDB, so I am biased.

One cool thing about Postgres is the ability to add a generated column. If you've decided to go schemaless, you can later generate a column from a field in your JSONB blob and treat it like any other column.
