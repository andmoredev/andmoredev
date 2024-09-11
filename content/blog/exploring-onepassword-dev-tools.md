+++
title = "Exploring 1Password Dev Tools"
date = 2024-08-26T00:00:00-00:00
draft = false
description = "People are usually surprised when I mention that 1Password has quite a bit of dev tools that can greatly improve how we manage secrets in the development lifecycle. In this post I want to walk through some of the features that 1Password provides on top of incredible password management capabilities."
tags = ["Security"]
[[images]]
  src = "img/exploring-one-password-dev-tools/title.png"
  alt = "Title of the blog post and user icons with two of them with a picture of Andres"
  stretch = "stretchH"
+++

We usually see 1Password and only think about their password management capabilities, which to be honest I love. But something that not a lot of people know is that they also provide tooling for engineers to access sensitive data with minimal effort. In this post I want to highlight some of the features 1Password provides for developers.

When you access the directory of developer tools they provide they have four main categories for what they offer.  

1. [Access Tokens](#access-tokens)
1. [Infrastructure Secrets Management](#infrastructure-secrets-management)
1. 1Password CLI
1. SSH & Git
1. IDE Extensions

![Developer Tools Directory List](/static/img/exploring-one-password-dev-tools/developer-tools-directory.png)

Before going into the details of each of these tools I want to explain how I have my secrets organized within 1Password so my examples make sense.  
I have a vault called *andmoredev-sandbox*. The intention is to have a vault per environment for my andmoredev projects. Below is a screenshot of my 1Password account and I've pointed out the three main pieces to understand.

1. The *andmoredev-sandbox* vault that I will be using for the examples.
1. This is one of the items of several contained within this vault. Here I have highlighted the *cicd* item that contains information that is needed within my CI/CD pipelines.
1. Within an item you can have different pieces of information like plain text fields, email, ont time passwords, phone, etc. In the example below I've highlighted a field that I have set as a password that contains the role needed by GitHub to be able to authenticate with AWS.

![andmoredev-sandbox 1Password Vault](/static/img/exploring-one-password-dev-tools/1password-overview.png)

Now that you can understand the structure of my vaults and secrets, let's dig into the different types of tools they provide.

## Access Tokens
They provide token-based authentication with service accounts to allow us to access our secrets securely programmatically.

To create access tokens you will need to go to the [developer tools directory](https://my.1password.com/developer-tools/directory) and sign in to your account. From there press the *Service Account* button and it will take you through a wizard to generate your token.

## Set Service Account name
There will be some structuring that you'll need to figure out on how you want to isolate your secrets. For my case I will separate them by environment and application, which ends up being "Sandbox - 1Password Tests"

![Setup your service account name](/static/img/exploring-one-password-dev-tools/setup-your-service-account.png)

## Grant vault access
Based on how you've structure your vaults and your secrets you will now be given the opportunity to grant access to specific vaults. I have setup a specific vault that contains all my secrets for my sandboxes (which is where I usually deploy all of my tutorials).
For each of the vaults you select you will be able to grant specific permissions for *read*, *write* or *share items. For most use cases you will only need to read secrets unless you are building some sort of administration application. Please make sure to follow the principle of least privilege when setting this up.

![Grant vault access](/static/img/exploring-one-password-dev-tools/grant-vault-access.png)

## Grab and save your service account token
The last step gives you the token for you to use. They have a button that allows you to very easily save that token within your 1Password account if you would like to keep track of it.

![Save service token](/static/img/exploring-one-password-dev-tools/save-service-token.png)

Now that we've set this up we can dig into the more fun stuff since we can now interact with 1Password directly with their other tools.

## Infrastructure Secrets Management
1Password has built in integrations with several DevOps platforms. What I've typically done to manage secrets and api keys is that I store them in a password manager to be able to share the values with the rest of the team. Then these secrets are copied into the DevOps platform as secrets and/or variables, now when you run your workflows you have access to these values. By integrating directly with 1Password you can skip the middle step and access the values directly from 1Password therefore reducing the amount of times you have to copy the sensitive information. By doing this you also reduce the amount of maintenance when having to rotate keys since you will only have to change them from 1Password instead of each repository that consumes it.

1Password has integrations with [CircleCI](https://circleci.com/), [GitHub Actions](https://docs.github.com/en/actions) and [Jenkins](https://www.jenkins.io/). If you use something else you can use the [1Password CLI](https://developer.1password.com/docs/cli/) which will be covered in a later section. Let's go over the supported integrations and how you can use these.

### GitHub Actions
For GitHub they have a GitHub Action named [load-secrets-action](https://github.com/marketplace/actions/load-secrets-from-1password), this takes care of the heavy lifting and all you have to do is provide your service token and the secret identifiers for the ones you want load.

In the example below I'm loading all the information I need to be able to successfully authenticate and deploy to AWS.  
```yaml
      - name: Load secret
        id: op-load-secret
        uses: 1password/load-secrets-action@v2
        with:
          export-env: true
        env:
          OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
          PIPELINE_EXECUTION_ROLE: op://andmoredev-sandbox/cicd/pipeline-execution-role-arn
          CLOUDFORMATION_EXECUTION_ROLE: op://andmoredev-sandbox/cicd/cloudformation-execution-role-arn
          ARTIFACTS_BUCKET_NAME: op://andmoredev-sandbox/cicd/artifact-bucket-name

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: ${{ env.PIPELINE_EXECUTION_ROLE }}
          role-session-name: sam-deploy
          role-duration-seconds: 3600
          role-skip-session-tagging: true
```

The easiest way I found to consume these is by setting the environment variables as seen above. We first set the *OP_SERVICE_ACCOUNT_TOKEN* which is a previously generated token with the necessary permissions to read from the 1Password vault. Then we are providing the identifiers for all the secrets we want to load, you can get these identifiers by going into the 1Password application and selecting the *Copy Secret Reference* option as shown below.

![Copy secret reference](/static/img/exploring-one-password-dev-tools/1password-copy-secret-reference.png)

The result will be a value on your clipboard that has the following format `op://[vault name]/[item name]/[secret name]`.

The only property you can provide to the GitHub action is the *export-env* boolean. This is specifying if you want the action to automatically set the response values into the environment to be used. Since I have set that to true, you can see how I'm referencing the secret directly from the env with `${{ env.PIPELINE_EXECUTION_ROLE }}`, if we had set that to false we would've had to reference the output like this `${{ steps.op-load-secrets.outputs.PIPELINE_EXECUTION_ROLE }}`. You can choose whatever option makes more sense to you, both of these make sure that the values are masked wherever they are printed within the workflow.

### CircleCI
1Password has a [CircleCI orb](https://circleci.com/developer/orbs/orb/onepassword/secrets) that works very similarly to 
