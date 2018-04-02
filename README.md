# NGramLab
In this lab we will be processing Google's NGram dataset, a free dataset on AWS that contains word pairs from the Google Books digitization project.  
[NGram Dataset Description](https://aws.amazon.com/datasets/google-books-ngrams/)

## Creating an Input Table
In order to process any data, we have to first define the source of the data. We do this using a CREATE TABLE statement. For the purpose of this example, I only care about English 1-grams. This is the statement I use to define that table.

'''sql
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
'''
