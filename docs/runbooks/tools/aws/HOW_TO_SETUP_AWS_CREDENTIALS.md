# How to Set Up the AWS Environment

Once you've been added to the AWS account, you should be able to log into the [lab AWS portal](https://nu-sso.awsapps.com/start/#/console?account_id=517478598677) using your Northwestern credentials.

To be able to work with AWS through code (e.g., accessing AWS resources in your Python scripts), you need to set up local credentials.

This can be done in one of two ways:

- A named profile (`AWS_PROFILE`): this is the recommended way.
- Access key pair: less recommended unless you know what you're doing.

## Approach 1: Setting up `AWS_PROFILE`

Setting up the `AWS_PROFILE` is the default and recommended way to access AWS locally.

1. Download the AWS CLI, using [this link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
2. Configure your AWS profile, one time, using `aws configure`.
3. Confirm that the profile works: `aws sts get-caller-identity`. You should see a JSON with `Account`, `Arn`, and `UserId`.
4. (Optional) in `.aws/config`, where you set your profile and can `cat` your profile info (see `cat .aws/config`), you should have a default region set. If not, then in the `.env` file of wherever you're using AWS, add fields for `AWS_PROFILE` and `AWS_REGION` as well.

## Approach 2: Setting up an access key pair

Setting up an access key pair is less recommended unless you're explicitly working with remote servers. If you decide to go the access key pair route, then if `AWS_PROFILE` is set, boto3 prefers the profile and ignores `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` even when those keys are also present. Also be sure to track what the key is doing and rotate on a reasonable cadence.

You'll need to set the following in your environmental variables:

```bash
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

To get these:

Step 1: Create an IAM user

- Go to "IAM" -> "IAM users"
- Create a new user
- Go to "Attach policies directly"
- If you want the policy to access any AWS resources, add "PowerUserAccess"

Step 2: Get IAM credentials

- Click into the user.
- Go to "Access Keys" and create a new one.
- Select "Command Line Interface (CLI)".
- Create a description tag.
- Record your access key ID and secret
