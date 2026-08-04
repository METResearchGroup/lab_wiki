# How to cluster text

## What is clustering and why do we do it?

Clustering is the process of grouping items so that items in the same group are more similar to each other than to items in other groups. Normally, this means something like documents, posts, responses, sentences, or phrases, represented in a way that lets us measure similarity.

We cluster text for a few recurring reasons in lab research:

- **Discovery without labels.** A lot of our corpora (social media, open-ended study feedback, free-text survey responses) don't have clean labels. Clustering is a way to surface themes or topics without needing a predefined taxonomy up front.
- **Turning raw text into features.** Clusters often become candidate features for downstream analysis or modeling. See [HOW_TO_MINE_TEXT_FOR_FEATURES.md](HOW_TO_MINE_TEXT_FOR_FEATURES.md).
- **Exploration before modeling.** Even when you already have a modeling goal, clustering is a strong first pass for understanding what is in the data, what is noisy, and what distinctions actually matter.
- **Compression and summarization.** Large text corpora are hard to read end-to-end. Clusters give you a smaller set of examples and group summaries to review instead of every individual document.

Clustering doesn't just work automatically or in a vacuum. It's dependent on the quality of data passed into it, assumptions about the clustering model, and a human interpretation of what groups the algorithms discover.

## Baseline naive clustering approaches

Prior to using any complicated clustering algorithms, it's always good to step back and try the naive versions first. This can take a variety of approaches, such as:

- Creating clusters of words that have positive/negative sentiment.
- Doing bag-of-words and creating clusters based on if a word is high/low frequency.

These can be OK as first pass baselines. They also do well for allowing you to explore the data, which is critical for any modeling or ML training.

## Setup

### Embedding generation

When we cluster, we want to generally cluster on **units of meaning**, not necessarily tokens. It's up to you to determine what this looks like for your particular exercise. This may include, for example, entire tweets, entire posts, entire user responses, or it could include some phrase-level or sentence-level combination thereof.

For an embedding generation model, a good simple one to use is `all-MiniLM-L6-v2` (available via HuggingFace and runnable locally). A slightly larger and more up-to-date one is Amazon's Titan embedding model series. The lab has access to AWS Bedrock, which lets you access ML/AI models (including embedding models).

```python
import boto3

def get_bedrock_embedding(text, model_id='amazon.titan-embed-text-v1', region='us-east-1'):
    """
    Generate an embedding for the given text using AWS Bedrock's embedding model.

    Args:
        text (str): The input text to embed.
        model_id (str): The embedding model id on Bedrock.
        region (str): AWS region.

    Returns:
        list: The embedding vector as a list of floats.
    """
    client = boto3.client('bedrock-runtime', region_name=region)
    response = client.invoke_model(
        modelId=model_id,
        body=json.dumps({"inputText": text}),
        accept="application/json",
        contentType="application/json"
    )
    response_body = response['body'].read()
    embedding = json.loads(response_body)['embedding']
    return embedding

# Example usage:
# text = "Clustering helps group similar documents."
# vector = get_bedrock_embedding(text)
# print(vector)
```

A decent baseline dimensionality to start at would be something like `dim=256`.

## Algorithm 1: K-Means

### What is K-Means?

K-Means partitions your items into a fixed number of groups, `k`. Each group has a centroid (the average embedding of the points in that group). The algorithm repeatedly assigns every point to the nearest centroid, then recomputes centroids, until assignments stabilize.

For text, you almost always run this on embeddings of your units of meaning (see Setup above), not on raw tokens. The output is a cluster ID per document/response/sentence, plus a centroid you can use to find representative examples.

#### What assumptions does the K-Means algorithm make?

- You choose `k` up front. The algorithm will always produce exactly `k` clusters.
- Clusters generally converge to be roughly spherical and similar in size in embedding space. This is because we cluster on distance, so if a given cluster would be too oblong or too large in diameter compared to other clusters, then it becomes increasingly likely that outer points will be assigned to other clusters. Consider this as you're interpreting the meaning of the clusters you find, as if you're expecting one cluster to have more samples than another, k-means might be a poor choice.
- Every point gets a cluster. There is no built-in "noise" or "outlier" bucket.
- Typically Euclidean distance is used, but you can create your own distance function. If you care more about direction than magnitude, L2-normalize embeddings first so Euclidean distance behaves more like cosine distance.
- Results are really dependent on initialization (since the algorithm initializes centroids at random). Whenever you're doing k-means, make sure that you add a random seed in order to ensure reproducibility.

#### K-Means algorithm pseudocode

Here's some pseudocode describing how K-Means works, for your intuition:

```text
Input: points X = {x_1, ..., x_n}, number of clusters k
Output: cluster assignment for each point, centroids C = {c_1, ..., c_k}

1. Initialize k centroids (e.g., randomly sample k points, or use k-means++)
2. Repeat until assignments stop changing (or max iterations reached):
   a. Assignment step:
      For each point x_i:
          assign x_i to the nearest centroid
          (argmin over j of distance(x_i, c_j))
   b. Update step:
      For each cluster j:
          set c_j = mean of all points assigned to cluster j
3. Return final assignments and centroids
```

### When to use/not use K-Means?

#### When to use K-Means

- You want a fast, simple first cut after naive baselines.
- You have a rough idea how many themes you expect (or are willing to try a small range of `k` to find some examples).
- Themes are fairly distinct and there's not a lot of crossover or blur.
- You mainly need groupings to inspect that you can then use for other downstream analysis.

K-means make for easily understandable clusters that can lead to interpretable and convincing visualizations in papers. However, beware as they may not converge to clean separable groups like you'd like. If you're luckiy, it might be as easy as varying the number of centroids `k`. Otherwise, you may want to investigatae other clustering algorithms.

#### When to possibly avoid K-Means

- `k` is genuinely unknown and hard to guess. If your task is difficult enough that sweeping a variety of `k` values doesn't work easily, it may be good to try other methods.
- You expect nested structure (e.g., "politics" splitting into "elections" vs "policy"). It's likely for such a scenario that, if you get a convergence, k-means will discover a general "politics" category.
- You need an explicit outlier/noise class for junk or off-topic text.

### Choosing 'k'

There is no single correct `k`. Treat it as a research choice, not a metric to optimize blindly.

Some thoughts:

- A quick win is to write a script that runs K-means clustering given `k` centroids, and then run that in a for-loop between some range (e.g., 2-10), and then visualize the clusters. Ideally you could visualize the clusters through some interactive dashboard in order to investigate the clusters yourself.
- Another approach is to try the grouping task yourself. Take a random subset of your corpus, say 20 samples, and try to put them into "distinct groups", however you define that. This should also tell you if there are certain groups that you'd like to emerge from your dataset, which can guide your approach.

### How to interpret clusters

For each cluster:

- Pull nearest-to-centroid examples. These are the "prototypical" examples of the cluster.
- Skim a random sample of members, not just the best exemplars.
- Check size of the cluster. A tiny cluster might be niche or may be noise, while a huge cluster might be too vague.
- As you look at it, see if you can determine a human-readable label for the cluster. You can also use an LLM for this task.

To see if this generalizes, take text outside of the sample used to build the clusters, embed it, assign it to a cluster, and see if the takeaways make sense.

### Where does K-Means go wrong?

- Forced partitioning means everything has to fit into a cluster: You might find that you end up creating a "miscellaneous" cluster from all the other text that don't fit cleanly into any other cluster.
- Bad `k`: Too small and you don't get distinct groups. Too big and you might make up clusters that aren't really unique.
- Sensitive to randomness: Different seeds can reshuffle boundaries. Try across a few seeds. If results change as you shuffle seeds, the clusters might not be stable.
- Representation mismatch: If embeddings don't capture the similarity you care about, no choice of `k` will help. First make sure that embeddings are the right modality for representing your data.
- Spherical assumption: If the true "ideal cluster" would be elongated or somewhat odd-shaped, K-means may not represent those well.

## Algorithm 2: BERTopic

BERTopic is ...

I highly recommend combining BERTopic with an LLM in order to generate human-readable descriptions of the topics, see [this guide](https://maartengr.github.io/BERTopic/getting_started/representation/llm.html) for more information.

### What is BERTopic?

#### What are the steps in BERTopic?

...

#### What problems with other methods does BERTopic try to solve?

...

#### What assumptions does BERTopic make?

#### Read this before using BERTopic

If you're planning on using BERTopic, I'd suggest starting with [this guide on BERTopic](https://maartengr.github.io/BERTopic/getting_started/quickstart/quickstart.html).

### When to use/not use BERTopic

#### When to use BERTopic

...

#### When not to use BERTopic

...

### Controlling the number of topics

...

### How to interpret topics

...


## Algorithm 3: Hierarchical clustering

...
