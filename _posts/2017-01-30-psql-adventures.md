---
layout: post
title:  "PSQL Adventures"
date:   2017-01-30
categories: jekyll update
---

Continuing on the [VoterKarma](http://voter-karma.herokuapp.com) project that I
began as part of a hackathon a few weeks back, I have been messing around with
the RDS database instance that we setup to store the voter file data.  In
particular, the original database only held about half of the voter data that I
had processed for election histories.  Once outside of the hackathon time
crunch, I wanted to go back and update the database with the missing values.
Unfortunately, I did not have a clear demarcation between records that had been
previously included in the database and those that had not.  Further, the
currently populated records were also linked to the `voter_grades` table through
their `id` column as a foreign key - so simply nuking the existing database and
uploading the total data set would not be ideal.  And so I started looking for
ways around this.

## Dumping Data from One DB to RDS
A quick aside, there are some nifty ways to move a lot of data from your local
database (or any db for that matter) and getting them into your RDS instance.
One way to do this quickly and modestly efficiently is to dump the data, gzip
it, push that zipped file to an ec2 instance, unzip on ec2, and finally insert
into RDS from ec2.  The benefit of this method is that it allows you to push the
compressed .gz file through `scp` rather than the uncompressed data file.
Depending on your internet connecton, this can be a very time-expensive step, so
reducing the package size is particularly helpful.

## Finding the New Rows
There is a pretty straightfoward way to find which rows in the "new" data set do
not already exist in the "existing" database.  In this case, one can just do a
LEFT OUTER JOIN from the "new" on the "existing" and condition it on rows where
the existing table's column `IS NULL`.  It would look something like this:

```SQL
SELECT *
FROM new_table
LEFT OUTER JOIN existing_table
USING(<join_column>)
WHERE existing_table.id IS NULL;
```

## Inserting New Rows
Finding these new rows is pretty easy, inserting them can be a little more of a
pain.  Notably, these new rows typically contain an id column from the
`new_table` that we don't want to include when inserting into the
`existing_table`.  Apparently, and unfortunately, there is no simple way to
SELECT all columns except for one in PSQL, so instead we need to use a kind of
complex query creating function.  Basically, it looks through all of the columns
and allows you to exclude a list that you do not want.

```SQL
SELECT 'SELECT ' || array_to_string(ARRAY(SELECT 'n' || '.' || c.column_name
        FROM information_schema.columns AS c
            WHERE table_name = 'new_table'
            AND  c.column_name NOT IN('id')
    ), ',') || ' FROM new_table AS n' AS sqlstmt

=> SELECT <lots_of_table_names> FROM new_table AS n
```

This SQL will just produce the query, with the long list of column names (my
table has about 50 columns, so it's not realistic to type them manually)
excluding, in this case, the id column.  There is probably a way to store this
query to be executed elsewhere, but I couldn't find an easy way of doing that
and instead just copy and pasted the output in my .psql file, which I use later
for my insert statement.

As a fun side note, I also discovered that I could use this pattern to create
dynamic select statements for my election columns (each of which begin `e20...`
followed by remaining dates and election indicators.  I could simply modify the
above sql to pattern match for these values, as such:


```SQL
SELECT 'SELECT ' || array_to_string(ARRAY(SELECT c.column_name::text
        FROM information_schema.columns AS c
            WHERE table_name = 'rawvoters'
            AND  c.column_name LIKE 'e20%'
    ), ',') || ' FROM rawvoters as r' AS sqlstmt

=> SELECT <lots_election_col_names> FROM rawvoters AS r
```

