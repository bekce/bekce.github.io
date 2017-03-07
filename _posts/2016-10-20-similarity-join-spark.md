---
layout:     post
title:      Similarity Join on Spark
date:       2016-10-20
categories: spark
---

Similarity Join is a widely used technique to find out the similarity of two (usually) string vectors like phrases, sentences or whole paragraphs of text. The basic idea is to build a metric to calculate the similarity score of each pair then if the value is within a certain threshold, output those pairs as similar with calculated score. The metric to build should be fairly dependent on the nature of vectors to compare, as there are multiple ways to compute similarity. I have implemented Similarity Join in Apache Spark using Jaccard similarity metric and count filtering, which reduces the runtime by reducing the number of comparisons to perform.

## Input format

`id\tcontent`

Example:  
`679097449584001024 Don't know when I should get my haircut`  
The id needs to be unique across inputs and is used for outputting.

## The algorithm explained

1. Tokenize the record, do appropriate data cleaning and get list of tokens, with their occurrence counts.
2. Contact the word to vector data set and get the semantically close words w.r.t. cosine similarity for each token on the list.
3. For each semantically close word, multiply its occurrence count with cosine similarity value and append it to list. Note that the occurrence value has become decimal.
4. At the end we will have a mixed list of tokens and counts, emit them as  
`key → (record, total_count, count)`  
`total_count` is the sum of all token counts for given record. We need this value in our last step, where we calculate the Jaccard similarity values.
5. Emit each record pair:  
`((record 1, total_count), (record 2, total_count)) → (key, count)`
6. Calculate similarity of each record pair w.r.t. Jaccard filtering and given threshold value. Jaccard count filtering simply selects pairs when this condition is met:  
`|a∩b| / |aub| >= t`

Pseudo-code for the similarity function:
    is_similar (r1, r2, threshold) : boolean
    define shared as number_of_shared_tokens_in(r1,r2)
    define similarity as (r1.total_count + r2.total_count - shared) / shared
    return similarity >= threshold

## Source Code

{% gist 95a3bc5397391b524a0296ee463796c2 %}

**Instructions**: just change the input and output paths and paste it into `spark-shell`.

To be improved.
