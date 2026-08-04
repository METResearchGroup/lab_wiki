# Some approaches for mining text for features

A common task that comes up during our research is mining a text corpus, whether it be social media content, feedback from study participants, or other modalities, and trying to extract common features. This is something that's done on a person-by-person basis and is often kept as tribal knowledge through experience. However, this guide attempts to write out some thoughts on approaches towards mining text.

## Approach 1: Classic extraction methods

A simple naive method to try is something like document frequency: in how many documents (e.g., in how many user responses) did a certain feature or word or term appear?

While doing this, some things to consider doing include:

- Lemmatize
- Remove stopwords and "unimportant words".
- Basic synonym merging: this can start as a first-pass set of hardcoded words. However, as you're doing clustering, you might notice more words that should be treated as synonyms, at which point you can expand the synonyms list. Err on the side of merging conservatively, as descriptions may be similar but represent slightly different concepts. Also prefer to use a hardcoded set of words rather than asking the LLM to do it at runtime, as a hardcoded set of words (even if it's a hardcoded set generated from asking an LLM "create a hardcoded set of synonyms") is better for auditing your work later.
- Domain-specifc noisy words: there are words that won't be filtered out through normal NLP methods, but we know are things that should be removed. Some words include "thinking", "keeping", "deciding", "response", etc. and depend on the task context. These can be created by hand by just reviewing the feedback manually and gathering a list (note: yes, this is the sort of thing that TF-IDF theoretically penalizes, but we don't want TF-IDF as we want common words used that are *not* these noisy words). Treat this as an experiment-specific stopwords list.

We want to avoid TF-IDF here actually as we want to maximize term frequency, which is corrected for in TF-IDF.

Another axis to explore is unigram vs. bigram vs. n-gram for extraction. Unigrams work OK as a start if you've done some basic cleaning as noted above.

This depends on your use case. For example:

- Unigrams may do well for features like `misinformation`, `triggering`, `violence`.
- Bigrams may do well for compounds like `civil engagement`, `political party`, and `free speech`.
- `n-grams` are more niche and likely not worth it unless you have a reason to extract a specific n-gram phrase.

Another fancier approach to consider is [RAKE](https://github.com/u-prashant/RAKE) (this is what ChatGPT pointed me towards when I asked about extraction methods). I'd err on the side of using classic unigram/bigram methods and avoid overcomplicating using less-supported methods like RAKE.

## Approach 2: Clustering

Clustering is another promising way to mine text for features. To see an overview of some clustering approaches you might want to consider, check out [CLUSTERING_ALGORITHMS.md](CLUSTERING_ALGORITHMS.md). A default one to try would be BERTopic, but please review how the algorithm works first so you know all the ways it might fail (and what are some possible alternatives).


## Approach 3: Asking an LLM to give features

...

An approach that I personally like is in [this GitHub README](https://github.com/docwriter-org/mine-writing-rules/tree/main#how-it-was-built).

### Some cons to consider

The con with this is if people cite multiple reasons in one. For example:

- "The level of vulgarity or the inappropriate writing style (e.g., ALL CAPS)."
- "The malice and lack of real thought behind the post"

These two examples show multiple reasons...

To counteract these .... (TODO: come up with something).

- Simplest approach: splitting on conjunctions
- Another approach: ask an LLM to rewrite comments

## Combining generated features using an LLM

Whether you use classical methods or clustering to get groups of possible features, you can often pair this with an LLM to generate human-readable labels. This isn't applicable for BERTopic (as LLM label generation is already done natively with BERTopic). The LLM can be tasked with taking the groups and creating a label for it.

A prompt can be something like:

```markdown
{SYSTEM PROMPT}

Group 1:
{Group 1}

Group 2:
{Group 2}

...
```

For example:

```markdown
You are helping label clusters of open-ended study feedback about why participants flagged social media posts.
For each group below, propose a short feature name (2–5 words) and a one-sentence definition.
Prefer features that could be coded as present/absent on a post. Do not invent themes that aren't supported by the terms.

Group 1:
Documents:
- "This post spreads false claims about the election with no sources."
- "Looks like fake news / conspiracy content meant to mislead people."
- "The statistics here are inaccurate and taken out of context."
Keywords: misinformation, false claim, fake news, misleading, inaccurate, conspiracy

Group 2:
Documents:
- "The post threatens people who disagree and feels aggressive."
- "Language that glorifies violence or harm toward a group."
- "Intimidating tone — reads like an attack, not a discussion."
Keywords: violence, threatening, aggressive, harm, attack, intimidation

Group 3:
Documents:
- "It's just partisan political party talking points, not real civil engagement."
- "People calling for censorship when this is free speech."
- "Very polarized / us-vs-them framing around politics."
Keywords: free speech, censorship, political party, civil engagement, polarized, partisan

Group 4:
Documents:
- "The level of vulgarity or the inappropriate writing style (e.g., ALL CAPS)."
- "Mostly swearing and rude hostility instead of an actual argument."
- "Tone is inappropriate — shouting in caps and insults."
Keywords: vulgarity, ALL CAPS, inappropriate tone, rude, swearing, hostility
```

Note: this is basically what the [LLM generation step of BERTopic already does](https://maartengr.github.io/BERTopic/getting_started/representation/llm.html), but we can repurpose that for more general feature generation.
