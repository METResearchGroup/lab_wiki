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

...

#### What assumptions does the K-Means algorithm make?

...

### When to use/not use K-Means?

...

### Choosing 'k'

...

### How to interpret clusters

...

### Where does K-Means go wrong?

...

## Algorithm 2: BERTopic

...

If you're planning on using BERTopic, I'd suggest starting with [this guide on BERTopic best practices](https://maartengr.github.io/BERTopic/getting_started/best_practices/best_practices.html).

## Algorithm 3: Hierarchical clustering

...
