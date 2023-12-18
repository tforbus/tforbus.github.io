---
title: "k-nearest neighbors with pgvector"
date: 2023-12-04T05:50:23-08:00
draft: true
url: "/posts/knn-pgvector"
---

Vector databases are so hot right now. I also wanted to explore using a vector database to learn more about them, and at some point probably build a service that utilizes one. Surprisingly, AWS doesn't have that many options. DocumentDB isn't a one-to-one drop in for MongoDB,and doesn't have a local emulator for development. AWS Keyspaces is just a managed version of Cassandra, but is only compatible with Cassandra 3.x, and vectors weren't introduced until 5.0. You can stream changes from DynamoDB to an OpenSearch cluster, but OpenSearch is expensive. It turns out Postgres has an extension called [pgvector](https://github.com/pgvector/pgvector) that allows vector similarity searching. Postgres runs in basically every big cloud provider, so that's kind of cool, too. I'm on a Postgres kick right now, so I'm moving forward with it for this exploration.

(Azure's ComosDB also seems like a good fit, honestly. It even has a [local emulator](https://learn.microsoft.com/en-us/azure/costmos-db/emulator)).

## Vectors
In a vector database, data is stored as a vector. A vector is just an array of numbers representing the properties of some *thing*.
If you have a function `vectorEncode` that looks like this:

{{<highlight go>}}
func vectorEncode(input string) []float32 {
    // do some encoding...
}
{{</highlight>}}

You can store that output to your database and operate on it in a way you couldn't have if it was just a string. An example of a function that does this is [TensorFlow Universal Sentence Encoder](https://www.tensorflow.org/hub/tutorials/semantic_similarity_with_tf_hub_universal_encoder). You can supply a sentence like "Live, laugh, love." and get a vector output of 512 floats.
```json
[
    -0.04382157325744629,  0.054702553898096085,  -0.02422274462878704,
   -0.012554816901683807, -0.025859476998448372,  -0.05978195369243622,
     0.05023908242583275,   0.05367206037044525,  -0.05983870476484299,
     ...
]
```


## Nearest neighbors
k-nearest neighbors (kNN) is an algorithm that uses proximity to classify or make predictions about the grouping of a piece of data. For classifying, think of marking an email spam or not spam. For predictions, think about calculating how much your home would sell for based on square footage. The key take away here is it uses proximity (or distance) to determine where a particular item belongs. Since we have a way to encode *things* into vectors, we can find their similarity by calculating the distance between two vectors. We can do this across all vectors, and then find all the nearest neighboring vectors (all *k* of them).

pgvector has three ways to compute distance:
* Euclidean distance
* Cosine similarity
* Inner product

and three different ways to compute nearest neighbors: 
* exact nearest neighbor 
* approximate nearest neighbor with IVFFlat
* approximate nearest neighbor with HNSW

There is a lot to cover here, but I'd like to narrow the focus down to computing approximate nearest neighbors with HNSW (Hieararchical Navigable Small World). In general, approximate nearest neighbor algorithms have worse recall than exact nearest neighbor, but are faster. HNSW is an interesting choice compared to IVFFlat because you can immediately create an index on your table; IVFFlat requires some amount of data before it can properly index.


## Hieararchical Navigable Small World



<!--

I don't currently know when to use which distance computation, but the most common one is Euclidean distance, so that's honestly a good enough starting point for me. I've used cosine similarity when tinkering with vectors of text, and that's also been fine. I have no idea about inner product.

Exact nearest neighbor will return the same output for the same input, but approximate nearest neighbor algorithms don't. The trade off is recall vs performance. FAISS has [guidelines](https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index) you can use to determine on which algorithm to use for approximate nearest neighbors. The pgvector docs also list a few tradeoffs. IVFFlat uses less memory than HNSW, but has worse recall. You can create an IVFFlat index after you've gotten to the point that the exact nearest neighbor search isn't performant anymore. HNSW uses more memory than IVFFlat, but you can also create the index when you have no data. There is a limitation on the number of dimensions your vector can have (2000 in pgvector).

I don't know much about this at all, so it's time to experiment.

## Experimenting
Let's figure out when exact nearest neighbor breaks. There are two dimensions to this problem: the size of the vectors, and the number of vectors. 
-->
