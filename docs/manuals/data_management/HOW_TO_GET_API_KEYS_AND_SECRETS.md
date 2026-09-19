# How to get API keys and secrets

To access things like OpenAI or other services, you often need API keys, also known as secrets. Normally, the way that we do this is to have a .env file that has all this information, and then we load it at the script level. But this leads to people having to pass around different versions of API keys.

People have their personal API keys versus ones that are used by the lab, and it's a whole complicated process of tracking API keys without accidentally leaking them. To consolidate this, we use an AWS service that just keeps all the API keys, and we just ask the AI to be able to fetch the API keys whenever needed. This is called AWS Secrets Manager (see [this writeup for more details](docs/manuals/tools/aws/WHAT_IS_AWS_SECRETS_MANAGER.md)).

## How to get API keys

First, ensure that you have access to AWS. See [this writeup](docs/manuals/tools/aws/HOW_TO_SETUP_AWS_CREDENTIALS.md) and confirm with Mark that you have access to AWS.

Then, ask your AI agent this prompt:

```markdown
Use the AWS CLI and my credentials to access our AWS account. Then look at the AWS Secrets Manager and explain to me in a table what secrets are available, and filter by the ones that are likely most relevant for the work I'm doing right now.

Report back to me a table with three columns:
1. The name of the AWS secret
2. The description and what it is most likely used for
3. A brief description as to whether it's likely useful for what I'm working on right now
```

To see how this works, check out this [recording](https://zoom.us/clips/share/mwIjBiMzRYKBq9ZKPdFYGQ).
