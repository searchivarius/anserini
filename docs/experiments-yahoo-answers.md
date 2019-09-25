# Anserini: BM25 Baselines on [Yahoo Answers Manner (L5)](https://webscope.sandbox.yahoo.com/catalog.php?datatype=l)

This page contains instructions for running experiments using Yahoo Answers collections.
It focuses on using a smaller **L5** (i.e., **Manner**) collection, but one can use this code to play with a larger Yahoo Answers **Comprehensive** collection,
which has the code **L6**. 
These collections can be obtained for **non-commercial** use by scientists from [Yahoo Webscope](https://webscope.sandbox.yahoo.com/catalog.php?datatype=l&guccounter=1).

## Data Preparation

We are going to use `yahoo-answers/` as the working directory.
First, we need to download the data file `manner.xml.bz2` and place it in the `yahoo-answers` directory.
The checksum of this file should be `60dad52cf2f88bc49bb738634448a59f`.

Next, we need to convert this file into jsonl files (which have one json object per line).
Because, Yahoo Answers collections do not have a set of designated queries, 
we select a sample of 5K random questions to play a role of queries.
The choice is controlled by a seed value:

```
python src/main/python/yahoo_answers/convert_collection_to_jsonl.py \
--random_seed 0 --query_sample_qty 5000 \
--collection_path yahoo-answers/manner.xml.bz2 \
--output_folder yahoo-answers/collection_jsonl
```

For the smaller Manner collection, the conversion should take about one minute. It generates one document jsonl file,
one queries file, and one qrel file. The latter indicates which documents are relevant for each of the sampled query.
In this experiment, we assume that every answer provided by a community user is a relevant document. If an
answer is selected as a best answer by community users, it has grade four. Other answers provided by 
users have grade three. Answers given to other questions are considered to be non-relevant.

We can now index these docs as a `JsonCollection` using Anserini:

```
sh ./target/appassembler/bin/IndexCollection \
-collection JsonCollection  \
-generator LuceneDocumentGenerator \
-threads 4 \
-input yahoo-answers/collection_jsonl/ \
-index yahoo-answers/lucene-index  \
-storePositions -storeDocvectors -storeRawDocs 
```

Upon completion (should take less than a minute), we should have an index with 819,603 documents.


## Retrieving and Evaluating 

We can now retrieve using a sampled set of queries:

```
sh ./target/appassembler/bin/SearchCollection \
-index yahoo-answers/lucene-index  \
-threads 4 \
-topicreader Webxml \
-topics yahoo-answers/collection_jsonl/queries.xml \
-output yahoo-answers/output.trec  \
-bm25 -hits 100
```

and evaluate them:

```
./eval/trec_eval.9.0.4/trec_eval -c -mrecall.1000 -mmap  \
yahoo-answers/collection_jsonl/qrels.tsv \
yahoo-answers/output.trec
```

For the default BM25 settings the results should be:
```
map                   	all	0.0985
recall_1000           	all	0.2364
```


