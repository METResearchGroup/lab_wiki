# How to cluster text

## What is clustering and why do we do it?

...

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

...

## Algorithm 2: BERTopic

...

## Algorithm 3: Hierarchical clustering

...
