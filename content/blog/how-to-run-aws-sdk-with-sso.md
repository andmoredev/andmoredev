+++
title = "How to run AWS SDK locally when using AWS SSO"
date = 2023-01-27T00:00:00Z
draft = true
description = "Find out the different ways you can leverage AWS SSO to run code using the AWS SDK"
tags = ["AWS", "SDK", "IAM", "SSO"]
[[images]]
  src = "img/how-to-run-aws-sdk-with-sso/01.jpg"
  alt = "Engineers working in laptops"
  stretch = "stretchH"
+++
When using Access Keys I got used to setting the default profile in my credentials file and it all just worked. When I began having to manage more accounts and switched to SSO updating the default profile becomes more of a pain. I will show a couple of ways you can easily authenticate the SDK when using named profiles.

We will be walking through the steps to authenticate the AWS SDK against a specific AWS Account using the following methods:
1. Setting Environment Variables
2. Getting Credentials in the code.

## Setting Environment Variables
The easiest way you can run the AWS SDK from your local machine is by setting the AWS_PROFILE environment variable (if not set it will default to the *default* profile).

In your terminal you can set the AWS_PROFILE environment variable so that when the SDK is used it will find the profile from the `~/.aws/config` file and use those credentials. In our case using AWS SSO you will have to be logged in by running `aws sso login --profile my-profile`

*I couldn't get it to use the default region from the *config* file, so you will also have to set the AWS_REGION environment variable to make this work.*

**How do you set Environment Variables?**  
Linux/macOS
```powershell
$ export AWS_PROFILE=my-profile
$ export AWS_REGION=us-east-1
```
Powershell
```powershell 
PS C:\> $Env:AWS_PROFILE="my-profile"
PS C:\> $Env:AWS_REGION="us-east-1"
```
Windows Command Prompt
```powershell 
C:\> setx AWS_PROFILE my-profile
C:\> setx AWS_REGION us-east-1
```

While being authenticated via SSO and having the necessary environment variables setup you can instantiate the client as shown below.
```javascript
  const { SecretsManagerClient } = require('@aws-sdk/client-secrets-manager');
  const secretsManagerClient = new SecretsManagerClient();
```

There are more Environment Variables that you can set in you terminal but as I mentioned, you can simplify your local execution by using the profile setup.

## Getting Credentials in code
Another way to provide the credentials to the SDK is by using the **credentials-provider** class from the SDK. This requires your code to be aware of any parameters you need to set.  
To make this work the same way as the Environment Variable route you will need to provide the *profile* as an input to your code.

As with the Environment Variables, I couldn't get it to use the default region set in the *config* file so you will also need to provide the region as an input.

Once you are signed in to a specific profile by doing `aws sso login --profile my-profile`, you can get the credentials using the `credential-providers` package as shown below. If using the *default* you can start the clients the same way as with Environment Variables.

```javascript 
  const { SecretsManagerClient } = require('@aws-sdk/client-secrets-manager');
  const { fromSSO } = require('@aws-sdk/credential-providers');
  const credentials = await fromSSO({
    profile: inputtedProfile
  })();
```

Now with the AwsCredentialIdentityProvider object you can instantiate any client by providing the **credentials** and the **region**, like this:
```javascript
  const secretsManagerClient = new SecretsManagerClient({
    credentials: credentials,
    region: inputtedRegion
  });
```

## Conclusion
Running the SDK from a local machine can come in handy for development so being able to properly and securely authenticate is very important.  
We went through a couple of ways you can authenticate code that is being executed locally, there are other ways you can authenticate, so if these options do not work for you do to organization security policies I recommend looking into the documentation or reach out to the community to find which one fits you best.