# What is AWS Bedrock?

AWS Bedrock is their AI/ML solution. It's the one-stop shop for all LLM/AI/ML model training, storage, inference, etc.

It's useful for things like:

1. **Using various LLMs** (AWS Bedrock hosts all LLMs, from GPT to Anthropic to Qwen/Deepseek/open-sourced models). Rather than 4-5 different API keys across models, you can have it all in one place.
2. **Privacy and safety** AWS Bedrock has AWS Bedrock supports a zero-data-retention (ZDR) policy ([see here for more](https://aws.amazon.com/blogs/security/enforce-zero-data-retention-on-amazon-bedrock-with-bedrock-projects-and-service-control-policies/)). What this means, in short, is that AWS Bedrock doesn't store any of your queries or requests whenever you run a model.
3. **Organization-level discounts** Education insituations like Northwestern get discounts and preferred treatment from AWS.
4. **Usability with other AWS tools** AWS Bedrock cleanly integrates with other tools like Sagemaker and S3, so AWS can be your one-stop shop for all ML research.

## What are some examples of what AWS Bedrock can do?

1. Running a prompt across a bunch of different models. See [this example](https://github.com/METResearchGroup/mind_technology_lab_experiments/pull/20)
2. Fine-tuning a model. See [this example](https://github.com/METResearchGroup/mirrorView-task/pull/54).
