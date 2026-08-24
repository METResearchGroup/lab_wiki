# How to calibrate LLM confidence

A common pattern that I see is to ask the LLM to give a score about how confident it is, or to give a result only if it's confident in what it's doing. However, this relies on a particular assumption about the LLM being aware of and able to properly report its own reliability. This does not work.

For example, [this paper out of Amazon](https://arxiv.org/pdf/2608.04899) shows that LLMs are bad at reporting their own confidence. Here's a direct quote from the abstract:

> For instance, Qwen3-
32B verbalizes only eight unique confidence
values on SST-2, with over half being exactly
95%—a pattern we observe consistently across
four datasets and two LLMs

LLMs are poor judges of their own confidence. Here are some alternatives to asking the LLM for its confidence:

1. **Create the criteria for what constitutes confidence, and then ask the LLM to judge itself on those criteria.** Define criteria yourself on what constitutes a confidence score. This can be a series of questions that you would ask a person to understand how confident or sure they are of their output.
2. **When asking an LLM to answer questions, err on the side of giving it binary yes/no questions**: LLMs are a lot better at answering yes/no questions than questions on a Likert scale or a confidence interval (lots of papers have found this, such as [this one](https://arxiv.org/html/2606.27226v1)).
3. **Evaluate the softmax probabilities**: If you're using an open source model, you can look at the last layer to look at the softmax probabilities. That should give you some sort of sense of the confidence of its output. This only works for open source models and if you have the confidence in your technical ability to evaluate the softmax probabilities.

When in doubt, try to do calibration and confidence yourself. Get a few results and hand-label how "sure" or "certain" the label seems, based on the input. Then use that as an example set that you can use to either prompt the LLM with few-shot examples or direct your next approach. [Here's an example of how to do that](https://www.databricks.com/blog/enhancing-llm-as-a-judge-with-grading-notes).
