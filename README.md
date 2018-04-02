# NGramLab
In this lab we will be processing Google's NGram dataset, a free dataset on AWS that contains word pairs from the Google Books digitization project.  
[NGram Dataset Description](https://aws.amazon.com/datasets/google-books-ngrams/)

## Creating an Input Table
In order to process any data, we have to first define the source of the data. We do this using a CREATE TABLE statement. For the purpose of this example, I only care about English 1-grams. This is the statement I use to define that table.

```sql
CREATE EXTERNAL TABLE english_1grams (
 gram string,
 year int,
 occurrences bigint,
 pages bigint,
 books bigint
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS SEQUENCEFILE
LOCATION 's3://datasets.elasticmapreduce/ngrams/books/20090715/eng-all/1gram/';
```
This statement creates an external table (one not managed by Hive) and defines five columns representing the fields in the dataset. We tell Hive that these fields are delimited by tabs and stored in a SequenceFile. Finally we tell it the location of the table.

## Normalizing the data
The data in its current form is very raw. It contains punctuation, numbers (typically years), and is case sensitive. For our use case we don't care about most of these and only want clean data that is nice to display. It is probably enough to filter out n-grams with certain non-alphabetical characters and then to lowercase everything.
First we want to create a table to store the results of the normalization. In the process, we can also drop unnecessary columns.

```sql
CREATE TABLE normalized (
 gram string,
 year int,
 occurrences bigint
);
```

Now we need to insert data into this table using a select query. We'll read from the raw data table and insert into this new table.

```sql
INSERT OVERWRITE TABLE normalized
SELECT
 lower(gram),
 year,
 occurrences
FROM
 english_1grams
WHERE
 year >= 1890 AND
 gram REGEXP "^[A-Za-z+'-]+$";
```

There are a couple of things I'd like to point out here. First, we are filtering out the data before 1890 because there is less of it therefore it is probably noisier.
Second, I'm lower casing the n-gram using the built-in lower() UDF. Finally, I'm using a regular expression to match n-grams that contain only certain characters.

## Word ratio by decade
Next we want to find the ratio of each word per decade. More books are printed over time, so every word has a tendency to have more occurrences in later decades. We only care about the relative usage of that word over time, so we want to ignore the change in size of corpus. This can be done by finding the ratio of occurrences of this word over the total number of occurrences of all words.
Let's first create a table to store this data. We'll be swapping the year field with a decade field and the occurrence field with a ratio field (now of type double).

```sql
CREATE TABLE by_decade (
 gram string,
 decade int,
 ratio double
);
```

The query we construct has to first calculate the total number of word occurrences by decade. Then we can join this data with the normalized table in order to calculate the usage ratio. Here is the query.

```sql
INSERT OVERWRITE TABLE by_decade
SELECT
 a.gram,
 b.decade,
 sum(a.occurrences) / b.total
FROM
 normalized a
JOIN ( 
 SELECT 
  substr(year, 0, 3) as decade, 
  sum(occurrences) as total
 FROM 
  normalized
 GROUP BY 
  substr(year, 0, 3)
) b
ON
 substr(a.year, 0, 3) = b.decade
GROUP BY
 a.gram,
 b.decade,
 b.total;
```

## Calculating changes per decade
Now that we have a normalized dataset by decade we can get down to calculating changes by decade. This can be achieved by joining the dataset on itself. We'll want to join rows where the n-grams are equal and the decade is off by one. This lets us compare ratios for a given n-gram from one decade to the next.

```sql
CREATE TABLE final (
 gram string,
 decade int,
 ratio double,
increase double
);


INSERT OVERWRITE TABLE final
SELECT
 a.gram as gram,
 a.decade as decade,
 a.ratio as ratio,
 a.ratio / b.ratio as increase
FROM 
 by_decade a 
JOIN 
 by_decade b
ON
 a.gram = b.gram and
 a.decade - 1 = b.decade
WHERE
 a.ratio > 0.000001 and
 a.decade >= 190
DISTRIBUTE BY
 decade
SORT BY
 decade ASC,
 increase DESC;
```
## Changes over time
Now to show how the 'top trending words' in each decade changed over time we need to construct a new table consisting of filtered words.  First we need to create a new intermediate table with just the top 10 words for each decade.

```sql
CREATE TABLE final2
(
 gram string,
 decade int,
 ratio double,
increase double
);


INSERT OVERWRITE TABLE final2 
SELECT
a.gram,
CONCAT(a.decade,0),
a.ratio,
a.increase
    FROM (
        SELECT b.gram,b.decade,b.ratio,b.increase, Rank() 
          over (Partition BY b.decade
                ORDER BY b.increase DESC) AS Rank
        FROM final b
        ) a WHERE Rank <= 10;
```

## Optional
Then we can make a final set of data with the changes in prevalence of each of the top 10 words in each decade over time.

```
CREATE TABLE final3
(
 gram string,
 decade int,
 ratio double,
increase double
);


INSERT OVERWRITE TABLE final3
SELECT
 a.gram as gram,
 a.decade as decade,
 a.ratio as ratio,
 a.ratio / b.ratio as increase
FROM 
 by_decade a 
JOIN 
 by_decade b
ON
 a.gram = b.gram and
 a.decade - 1 = b.decade
LEFT SEMI JOIN final2 on (a.gram = final2.gram)
DISTRIBUTE BY
 decade
SORT BY
 gram ASC,
decade ASC;
```
