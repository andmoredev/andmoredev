+++
title = "Running AWS SDK locally"
date = 2023-01-31T00:00:00Z
draft = false
description = "Lear how to setup locally to run the AWS SDK using SSO and named profiles"
tags = ["AWS", "SDK", "IAM", "SSO"]
[[images]]
  src = "img/how-to-run-aws-sdk-with-sso/01.jpg"
  alt = "Engineers working in laptops"
  stretch = "stretchH"
+++
When using Access Keys I got used to setting the default profile in my credentials file and it worked just fine. When I began having to manage more accounts and switched to SSO, updating the default profile becomes more of a pain. I will show you a couple of ways to authenticate the SDK using named profiles.

I will be showing how to authenticate the AWS SDK against a specific AWS Account using the following methods:
1. Setting environment variables
2. Getting credentials in the code.

## Setting environment variables
The easiest way you can run the AWS SDK from your local machine is by setting the AWS_PROFILE environment variable (if not set it will default to the *default* profile).

In your terminal you can set the AWS_PROFILE environment variable so the SDK can find the profile from the `~/.aws/config` file and use those values. When using AWS SSO you will need to log in by running `aws sso login --profile my-profile`

*I couldn't get it to use the default region from the *config* file, so you will also have to set the AWS_REGION environment variable to make this work.*

**How do you set environment variables?**  
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

You can now use any client by initializing it as shown below.
```javascript
  const { SecretsManagerClient } = require('@aws-sdk/client-secrets-manager');
  const secretsManagerClient = new SecretsManagerClient();
```

There are more environment variables that you can set in you terminal to accomplish different things, but I wasn't able to find a consolidated list of the ones that are available to use.

## Getting credentials in code
Another way to provide the credentials to the SDK is by using the **credentials-provider** class. This requires your code to be aware of any parameters you need to set.  
To make this work the same way as the environment variable route you will need to provide the *profile* as an input in your code.

*As with the environment variables, I couldn't get it to use the default region set in the *config* file so you will also need to provide the region as an input.*

Once you signed in to a specific profile by doing `aws sso login --profile my-profile`, you can get the credentials using the `credential-providers` package as shown below (If using the *default* profile you can start the client the same way than with environment variables)

```javascript 
  const { SecretsManagerClient } = require('@aws-sdk/client-secrets-manager');
  const { fromSSO } = require('@aws-sdk/credential-providers');
  const credentials = await fromSSO({
    profile: inputtedProfile
  })();
```

With the AwsCredentialIdentityProvider object you can initialize any client by providing the **credentials** and the **region** as shown below:
```javascript
  const secretsManagerClient = new SecretsManagerClient({
    credentials: credentials,
    region: inputtedRegion
  });
```

## Conclusion
Running the SDK from your local machine is very important and valuable to be able to provide tooling and improve your developers experience.  
We went through some of the ways you can authenticate code that is being executed locally when you have several profiles  .
There are other ways you can authenticate, so if these options do not work for you do to organization security policies or any other reason, I recommend looking into the documentation or reach out to the community to find which one fits you best.