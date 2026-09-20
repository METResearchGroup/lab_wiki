# How to cluster text

## What is clustering and why do we do it?

Clustering is the process of grouping items so that items in the same group are more similar to each other than to items in other groups. Normally, this means something like documents, posts, responses, sentences, or phrases, represented in a way that lets us measure similarity.

We cluster text for a few recurring reasons in lab research:

- **Discovery without labels.** A lot of our corpora (social media, open-ended study feedback, free-text survey responses) don't have clean labels. Clustering is a way to surface themes or topics without needing a predefined taxonomy up front.
- **Turning raw text into features.** Clusters often become candidate features for downstream analysis or modeling. See [HOW_TO_MINE_TEXT_FOR_FEATURES.md](HOW_TO_MINE_TEXT_FOR_FEATURES.md).
- **Exploration before modeling.** Even when you already have a modeling goal, clustering is a strong first pass for understanding what is in the data, what is noisy, and what distinctions actually matter.
- **Compression and summarization.** Large text corpora are hard to read end-to-end. Clusters give you a smaller set of examples and group summaries to review instead of every individual document.

Clustering doesn't just work automatically or in a vacuum. It's dependent on the quality of data passed into it, assumptions about the clustering model, and a human interpretation of what groups the algorithms discover.

## Practical advice

If in a crunch, I suggest using **BERTopic** as the best all-in-one clustering approach. It combines clustering as well as generating human-readable labels for the clusters (so, it's more of a "clustering pipeline" rather than a "clustering algorithm"). However, it's good to know what the alternative approaches are, as:

1. Your task may not actually be a good fit for BERTopic.
2. It may not be clear how much of an improvement the BERTopic fit is unless you have simpler algorithms as a baseline.
3. You may want to swap out parts of the BERTopic algorithm for other approaches (e.g., replacing HDBSCAN with hierarchical clustering).

For using AI agents, I suggest asking them to pull the latest docs on BERTopic and proposing a plan. Make sure that you understand the basic steps of BERTopic (outlined below) and that the AI agent's build plan matches it. There are multiple distinct parts of BERTopic, and it's up to you to understand how each one fits together to generate the topics created by BERTopic.

In practice, I highly recommend combining BERTopic with an LLM in order to generate human-readable descriptions of the topics, see [this guide](https://maartengr.github.io/BERTopic/getting_started/representation/llm.html) for more information.

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

BERTopic is a topic modeling method that treats topic discovery as a pipeline. That pipeline has the following steps:

1. Embed documents: Convert documents to high-dimensional embeddings using transformer models like BERT.
2. Reduce dimensionality: Apply UMAP to reduce embeddings to 2D or 3D for effective clustering.
3. Cluster the reduced embeddings: Use HDBSCAN or similar algorithms to group similar documents.
4. Extract topic representations: Generate topic labels by identifying the most important words in each cluster.
5. Fine-tune topic assignments (optional): Optionally refine topic assignments or merge similar topics.

In practice, I highly recommend combining BERTopic with an LLM in order to generate human-readable descriptions of the topics, see [this guide](https://maartengr.github.io/BERTopic/getting_started/representation/llm.html) for more information.

Since BERTopic is a pipeline, you can swap different pieces (different embedders, different settings for UMAP, different clustering models, different representation models). The default parameters and setup work well out of the box though.

Another mental model to remember is that BERTopic collapses the task of feature discovery into basically two tasks:

1. Generate groups of similar items (via clustering).
2. Generate descriptions of each group (via topic representation).

K-Means can be used within BERTopic (especially in lieu of HDBSCAN). In addition, BERTopic, unlike K-Means, often includes a topic specifically for outliers, so outliers are less likely to pollute the generated groups. This is because HDBSCAN, unlike K-Means, is a density algorithm, rather than a distance algorithm, so it surfaces regions of high density rather than trying to fit centroids.

### What is BERTopic?

#### What are the steps in BERTopic?

At a high level, the default pipeline is:

```text
Input: documents D = {d_1, ..., d_n}
Output: topic ID per document (often including -1 for outliers), plus a representation per topic

1. Embed each document into a vector (sentence-transformers, Titan, etc.)
2. Reduce dimensionality (default: UMAP)
3. Cluster the reduced embeddings (default: HDBSCAN)
   - Dense regions become topics
   - Sparse points can be labeled as outliers (-1)
4. For each topic (cluster), build a representation of what it is "about"
   - Default: c-TF-IDF over the documents in that topic -> ranked keywords
   - Optional: KeyBERT-style keywords, MMR diversification, LLM labels, etc.
5. Return topic assignments + topic representations
```

#### What problems with other methods does BERTopic try to solve?

Relative to naive baselines and plain K-Means on embeddings, BERTopic addresses a few common pain points:

- Unknown number of topics: K-Means forces you to pick `k`. BERTopic's default HDBSCAN lets topic count emerge from density (controlled indirectly via things like `min_cluster_size`).
- Forced assignment of junk: K-Means puts every point somewhere. HDBSCAN can leave low-density items as outliers (`-1`) instead of polluting a "misc" theme.
- Topics without labels: K-Means gives you cluster IDs and centroids. BERTopic also produces keyword (and optionally LLM) descriptions so you can skim what a topic might mean before reading dozens of documents. You could do this with K-Means, but as a follow-up step.
- High-dimensional embedding geometry: Clustering raw high-dim embeddings can be brittle. UMAP reduction is meant to make neighborhood structure easier for the clusterer. But this also assumes that UMAP finds a decent 2D or 3D representation, which is in itself an assumption you should check.
- Classical topic models' bag-of-words limits: Methods like LDA work from word counts and often struggle with short, messy social/feedback text. BERTopic starts from semantic embeddings, which usually fit lab corpora better.

#### What assumptions does BERTopic make?

- The reduced embedding space has clear dense regions and that those dense regions correspond to actual themes. This is due to the assumptions of the HDBSCAN clustering algorithm, and if for your data this wouldn't work, then you can swap HDBSCAN with another clustering algorithm.
- You don't choose an exact topic count. Typically, you decide other hyperparameters (e.g., "how big can a cluster be?") and then based on this, a certain number of topics will emerge. Once again, this is specifically a parameter of HDBSCAN, so change accordingly if you swap for another clustering algorithm.
- Outliers are expected. A large `-1` bucket can be a feature (noise separation) or a bug (too aggressive clustering).
- UMAP (the default dimensionality reduction algorithm) is stochastic unless you set a seed with `random_state`. Without a seed, reruns can reshuffle topics.
- Each document gets a single topic. If your document has multiple themes, BERTopic assigns it to a single topic. To get around this, if you know a document will have multiple topics, you can split it into multiple documents beforehand.
- c-TF-IDF uses basic token counts to extract keywords for a given cluster. It assumes a reasonable tokenizer/vectorizer (language, stopwords, n-grams). Some of the caveats from the "## Approach 1: Classic extraction methods" in [HOW_TO_MINE_TEXT_FOR_FEATURES](HOW_TO_MINE_TEXT_FOR_FEATURES.md) apply.
- Treat cluster quality and representation quality differently. An LLM can generate feasible labels for a poorly generated cluster.

#### Read this before using BERTopic

If you're planning on using BERTopic, I'd suggest starting with [this guide on BERTopic](https://maartengr.github.io/BERTopic/getting_started/quickstart/quickstart.html).

### When to use/not use BERTopic

#### When to use BERTopic

- You do not have a reliable guess for `k`, or sweeping K-Means `k` did not yield stable, readable topics.
- You want an outlier bucket for off-topic / junk / one-off responses instead of forcing them into themes.
- You want keyword or LLM topic descriptions as a first pass before turning groups into features (see [HOW_TO_MINE_TEXT_FOR_FEATURES.md](HOW_TO_MINE_TEXT_FOR_FEATURES.md)).
- Your corpus is large enough that density-based clustering can find structure. Small corpora may overfit or may not converge.
- Topics may be uneven in size. Since HDBSCAN focuses on density, they are able to recover topics even if the counts of items in each topic aren't equal.

#### When not to use BERTopic

- You need a simple, fixed partition of exactly `k` groups for a clean figure or a tightly controlled coding scheme — K-Means is often easier to explain and control.
- Your dataset is tiny (dozens of documents). Density clustering and UMAP can be unstable or overconfident on small `n`.

### How to interpret topics

For each topic:

- Start with `get_topic_info()` / top keywords.
- Read representative documents for the topic, plus a random sample of members.
- Inspect the `-1` outlier topic separately. Ask whether those items are true noise, a missing theme, or a sign that `min_cluster_size` is too high/low. If you can find reasonable groups in the `-1` outlier topic, that might be a sign to try again.
- Check sizes. A swarm of tiny topics might mean too much fragmentation, while one giant topic might mean that you didn't generate cleanly separable topics.
- Draft a human-readable label (by hand or LLM), then verify it against held-out documents assigned to that topic.
- Separate errors by whether they're in (1) grouping or (2) representation: if you have feasible clusters but the labels generated for the cluster are poor, fix the representation layer. Otherwise, if the labels seem reasonable but the clusters are poor, address UMAP or HDBSCAN.

To check generalization, run `.transform` on text held out from fitting and see whether assignments still make sense.

### Best practices around BERTopic

These are covered in detail in [this link](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html) However, to make it explicit:

- [Precompute your embeddings](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html#pre-calculate-embeddings)
- [Add a seed to prevent randomness](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html#preventing-stochastic-behavior)
- [Control the number of topics](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html#controlling-number-of-topics)
- [Make tweaks to the default tokenizer to improve representations](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html#controlling-number-of-topics)

Point any AI agents to [the link](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html) before developing any BERTopic models.

## Algorithm 3: Hierarchical clustering

Hierarchical clustering builds a tree of how data points group together. It lets you customize where you want to "cut" or split groups in order to generate groups.

You typically start with each document as its own cluster and repeatedly merge the most similar clusters until everything is connected. The result is a dendrogram you can cut at different heights to get coarser or finer themes. Hierarchical clustering is useful when you expect nested structure (e.g., "politics" splitting into "elections" vs "policy").

This is a pretty niche application, but worth considering if you think there's some hierarchical structure in the sort of topics you'd like to uncover.

We'll keep the discussion here brief, and you can look up details for more information if it's applicable for your use case. For most of the lab use cases, either K-Means or BERTopic generally suffice.

### What is hierarchical clustering?

#### What are the steps? / algorithm pseudocode

The version you will almost always use is agglomerative (bottom-up):

```text
Input: points X = {x_1, ..., x_n}, a distance metric, a linkage rule
Output: a merge tree (dendrogram); optionally a flat cluster labeling after a cut

1. Start with n clusters (each point is its own cluster)
2. While more than one cluster remains:
   a. Find the two clusters that are closest under the chosen linkage
   b. Merge them into a new cluster
   c. Record the merge and the distance at which it happened
3. Return the full tree
4. (Optional) Cut the tree at a height or at a target number of clusters
   to get a flat assignment for downstream use
```

Unlike K-Means, you do not re-fit from scratch to try a different number of groups. You can cut the same tree at a different level to see what different groups emerge.

#### Linkage criteria

Linkage defines how you measure distance **between clusters** (not just between points). This is the important knob — analogous to choosing `k` in K-Means or `min_cluster_size` in BERTopic/HDBSCAN.

Common options:

- Single linkage: distance = minimum distance between any pair of points across clusters. Can create long "chains" of loosely related items.
- Complete linkage: distance = maximum pairwise distance across clusters. Tends toward more compact clusters; can be sensitive to outliers.
- Average linkage: distance = average pairwise distance across clusters. Often a solid default for text embeddings.
- Ward: merges the pair that least increases within-cluster variance. Often works well when clusters are roughly spherical in embedding space; typically used with Euclidean distance.

### Choosing where to cut the tree

There is no single correct cut. Treat it like choosing `k` in K-means clustering:

Some ideas include:

- Cut to a target number of clusters if you already have an amount in mind (e.g., 8)
- Cut by merge height / distance: look for a large jump in the dendrogram (a gap where very different clusters finally merge).
- Manually check like you would in K-Means: hand-group ~20 random samples, then see which cut recovers distinctions you care about.

## Notes and things to keep in mind

- Whatever results you get, make sure that you review it! The choices of features you get are closely tied to your hyperparameter choice, the prompts you give to the LLMs, the settings on your HDBSCAN, etc.
- HDBSCAN collapses across a 256-D embedding, so it often can miss groupings that a human annotator would've gotten. I recommend you review the "noise" cluster (the `-1` group) to see if it really is noise or if there are patterns within it. I also suggest that you ask an LLM to possibly recode the text and see what it generates, and use that to complement the groups that HDBSCAN finds.
- The clusters that an LLM finds are often more rich and diverse in my experience, but also can be dependent on your prompting. I suggest steering the LLM to look for a specific set of plausible features (e.g., semantic, lexical, etc.). See [this prompt](https://github.com/METResearchGroup/mirrorView-task/blob/main/experiments/create_llm_features_2026_08_05/src/prompts.py) for an example.
- The groups generated by each of these methods will likely differ, so be sure to know each algorithm's assumptions and feel free to try multiple approaches (or multiple parameters of the same approach) until you get a reasonable result.
