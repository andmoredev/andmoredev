+++
title = "Using Amazon Cognito with the user-password flow"
date = 2024-07-31T00:00:00-00:00
draft = false
description = "In May I released a post on how to secure APIs using machine-to-machine authentication. Exactly one day after that AWS Cognito changed their pricing model and now my proposed solution would generate cost for me. In this post I will go through a different setup using the user-password auth flow. This will still allow us to authenticate from automations and from Postman while keeping us in the free tier."
tags = ["AWS", "Security", "SAM"]
[[images]]
  src = "img/api-cognito-user-password/title.png"
  alt = ""
  stretch = "stretchH"
+++

On my post called [Secure API Gateway with Amazon Cognito using SAM](https://www.andmore.dev/blog/api-cognito/) I talked about different Auth terms and walked through a setup to use the [Client Credentials Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow), but Cognito recently introduced [pricing changes for machine-to-machine authentication](https://aws.amazon.com/about-aws/whats-new/2024/05/amazon-cognito-tiered-pricing-m2m-usage/) that will make this cost us and my main goal is to do this while staying in the free tier for personal projects that will not be generating any income. That is why in this post I am going to setup Amazon Cognito using a different flow called user password-based authentication. With this type of authentication we are charged based on the Monthly Active Users (MAUs) and AWS gives you the first 50,000 MAUs for free, and in my case this will usually stay at 1 per project, so I should be fine.

## How does the user-password flow work?
Initially I thought I could use [Basic Auth](https://en.wikipedia.org/wiki/Basic_access_authentication) where you provide the encoded username and password in the *Authorization* header but that is not the case. In the image below I have a picture that shows the interactions that happen to get an authenticated request using this flow.

![Authentication flow interactions](/img/api-cognito-user-password/USER_PASSWORD_AUTH-flow-summary.png)

Let's dive deeper into each interaction
1. Caller makes a request with the username and password to your API.
1. API makes a call to Cognito to get the Access, Id and refresh tokens and returns it to the user
1. User provides Id token in Authorization header with each API call.
1. API verifies token with Cognito to make sure it's valid.


## Setting it all up with SAM
We are going to stick with the same architecture for the Amazon Cognito resources to simplify each APIs configuration and to be able to manage users from a single location.

![CloudFormation Stack Architecture](/img/api-cognito-user-password/stack-architecture.png)

Now let's see how each of these pieces are set up using SAM.

### Auth Stack
This stack has the resources that will be consumed by our API stacks.
Full auth stack setup can be found [in this GitHub repository](https://github.com/andmoredev/cognito-auth)

#### 1. Updates to the User Pool
Unlike our M2M setup that didn't need any properties for the user pool, for this type of authentication we need to setup properties that are specifying the attributes to be hosted for the user such as email, names, etc.
```diff
  CognitoUserPool:
    Type: AWS::Cognito::UserPool
+   Properties:
+     UsernameAttributes:
+       - email
+     AutoVerifiedAttributes:
+       - email
+     VerificationMessageTemplate:
+       DefaultEmailOption: CONFIRM_WITH_LINK
+     EmailConfiguration:
+       EmailSendingAccount: COGNITO_DEFAULT
```

The user pool attributes are:
* **UsernameAttributes** - this is specifying what you allow as a user name. The options here are *email* or *phone_number*.
* **AutoVerifiedAttributes** - attributes that you allow to be verified automatically by Cognito. For example we are using *email* this means Cognito will automatically send a verification email to the user. If we didn't set this attribute an administrator would have to manually verify users in Cognito.
* **VerificationMessageTemplate** - This is where you can set your own template for the email that will get sent to users to verify. For this example we are using a default option provided by Cognito where they will confirm by clicking a link. The other option is *CONFIRM_WITH_CODE* where a user will receive a code and they have to enter it manually to verify.
* **EmailConfiguration** - This property allows us to setup the configuration for the sender email for the verification or any other communications happening from Cognito. In our case we are using *COGNITO_DEFAULT* which reduces the amount of setup we need to get an Amazon SES verified email. The *COGNITO_DEFAULT* has some [limits](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html) that you will need to consider if you are using it.

### API Stack
Our API Stack is simplified when using this flow. Why? We do not need to create a resource server since we will not be using OAuth capabilities. 

#### 1. Delete *UserPoolResourceServer*
As mentioned before, we don't need this resource anymore. So let's get rid of it from our stack by removing it from the template.

#### 2. User Pool Client Updates
Below is the definition for our user pool client.
```diff
  CognitoUserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      UserPoolId: !Ref CognitoUserPoolId
-     GenerateSecret: true
-     AllowedOAuthFlows:
-       - client_credentials
-     AllowedOAuthScopes:
-       - layerless-esbuild/api
-     AllowedOAuthFlowsUserPoolClient: true
+     SupportedIdentityProviders:
+       - COGNITO
+     ExplicitAuthFlows:
+       - ALLOW_USER_PASSWORD_AUTH
+       - ALLOW_REFRESH_TOKEN_AUTH
```

A few changes need to be done in order for this authentication flow to work. 
* **GenerateSecret** - We first get rid of this property since it is not needed for this flow.
* **AllowedOAuthFlows** - When using the User/Password auth flow we do not need to set this property.
* **AllowedOAuthScopes** - Since we are not using an OAuth flow we do not need to set up scopes.
* **SupportedIdentityProviders** - We are going to use *COGNITO* as our only provider. You can configure different identity providers to simplify your users sign in by using Google, Facebook or any of the supported providers.
* **ExplicitAuthFlows**: this is where we will configure the *ALLOW_USER_PASSWORD_AUTH* which will allow us to authenticate using a username and a password. I've added *ALLOW_REFRESH_TOKEN_AUTH* since it's required but we will not be going to do token refreshes in this post.

#### 4. Updates to API Gateway
The only thing that needs to change in the API Gateway is the removal of the *AuthorizationScopes*. For this flow this is not needed.
```diff
  Auth:
    DefaultAuthorizer: ClientCognitoAuthorizer
    Authorizers:
      ClientCognitoAuthorizer:
        UserPoolArn: !Ref CognitoUserPoolArn
-       AuthorizationScopes:
-         - layerless-esbuild/echo
```

THAT'S IT!! We have now successfully setup our API to authenticate using the user-password flow.

You can find the full working example in this [GitHub repository](https://github.com/andmoredev/api-gateway-auth-with-cognito)

## Testing with Postman

### 1. Grab Client Id
We only need the client id when authenticating with the user-password flow, this is because we are going to enter a username and password to authenticate and that is where the token will be coming from. So we will get this value from the console by going into our new application client.

![AWS Console showing an application client in Cognito where we can grab the client id](/img/api-cognito-user-password/cognito-client-keys.png)

### 2. Requesting auth tokens
To request the tokens we need to call Amazon Cognito specifying we are doing an *InitiateAuth* command.
This will require us to make a POST call to *https://cognito-idp.us-east-1.amazonaws.com*. The region will change depending on where you created your User Pool. We specify the command by adding the *X-Amz-Target*, we also need to specify the *Content-Type* since it's an AWS specific type. Below is a screenshot that shows how the headers should look.

![Postman request headers](/img/api-cognito-user-password/postman-headers.png)

The body requires the following parameters:
* **AuthFlow** - This is where we specify that we want to use the user-password flow by setting a value of *USER_PASSWORD_AUTH*
* **ClientId** - We will set the value to the one we grabbed in step 1.
* **AuthParameters** - in the object we will provide the *USERNAME* and the *PASSWORD* of our user.

![Postman request body](/img/api-cognito-user-password/postman-body.png)

When you send this request you should get a response back with the *AccessToken*, *IdToken*, *RefreshToken*, *TokenType* and *ExpiresIn*. This also returns the *ChallengeParameters* but this flow does not require any challenge responses to be generated. (I've edited the full response from the picture to not expose the full keys).

![Postman response body](/img/api-cognito-user-password/postman-response.png)

### 5. Make request
Grabbing the *IdToken* from the response we got from the request above we can now take that and use it to authorize the request to the API endpoint as seen below.

![Postman auth config and successful request](/img/api-cognito-user-password/postman-authorization-configuration.png)

## Authenticating automation calls
Whenever I change authentication methods I get stuck setting it all up for my test automation. This requires us to do the same thing we did in Postman but programmatically. I am doing this by running a small NodeJS script that will make the call as shown below.

```javascript
const initiateAuthResponse = await axios({
    url: process.env.COGNITO_URL,
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-amz-json-1.1',
      'X-Amz-Target': 'AWSCognitoIdentityProviderService.InitiateAuth'
    },
    data: JSON.stringify({
      ClientId: process.env.CLIENT_ID,
      AuthFlow: 'USER_PASSWORD_AUTH',
      AuthParameters: {
        USERNAME: process.env.USERNAME,
        PASSWORD: process.env.PASSWORD
      }
    })
  });


  const token = initiateAuthResponse.data.AuthenticationResult.IdToken;
```

If you look at the code above it should seem very familiar to what we did in Postman. Making a POST request with the necessary headers and body. Once you get the response you can do whatever you need with the tokens.

I've provided 2 examples on how to handle the user credentials for this script to use within a CI/CD pipeline.
1. Create user programmatically - In this example we are [creating a Cognito user programmatically](https://www.andmore.dev/blog/create-cognito-user-programatically/) and using the credentials to authenticate. We do this by setting the values as environment variables using the *>> $GITHUB_ENV* command. At the end of the run we delete the user so we don't have an endless amount of orphaned automation users.
```yaml
  test-api-with-user-password-auth-inline-create-user:
    name: Run Portman With USER_PASSWORD_AUTH - Inline Create User
    runs-on: ubuntu-latest
    environment: ${{ inputs.ENVIRONMENT }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: ${{ secrets.PIPELINE_EXECUTION_ROLE }}
          role-session-name: create-test-user
          role-duration-seconds: 3600
          role-skip-session-tagging: true

      - name: Create Test User
        run: |
          username=$(uuidgen)@andmore.dev
          password=$(uuidgen)G1%
          echo "USERNAME=$username" >> $GITHUB_ENV;
          echo "PASSWORD=$password" >> $GITHUB_ENV;
          aws cognito-idp admin-create-user --user-pool-id ${{ inputs.COGNITO_USER_POOL_ID }} --username $username --message-action SUPPRESS
          aws cognito-idp admin-set-user-password --user-pool-id ${{ inputs.COGNITO_USER_POOL_ID }} --username $username  --password $password --permanent

      - name: Test API
        env:
          COGNITO_URL: ${{ secrets.COGNITO_URL }}
          CLIENT_ID: ${{ secrets.COGNITO_CLIENT_ID }}
        run: |
          npm ci

          node ./portman/get-auth-token/user-password-auth.mjs
          npx @apideck/portman --cliOptionsFile portman/portman-cli.json --baseUrl ${{ inputs.BASE_URL }}

      - name: Delete Test User
        run: |
          aws cognito-idp admin-delete-user --user-pool-id ${{ inputs.COGNITO_USER_POOL_ID }} --username $USERNAME
```

1. Use GitHub Secrets - 
This is one of the simpler routes but requires to have a user already set up. All you need to do is store the credentials in GitHub secrets and reference them in the workflow.
```yaml
  test-api-with-user-password-auth-github-secrets:
    name: Run Portman With USER_PASSWORD_AUTH - Load GitHub Secrets
    runs-on: ubuntu-latest
    environment: ${{ inputs.ENVIRONMENT }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: ${{ secrets.PIPELINE_EXECUTION_ROLE }}
          role-session-name: create-test-user
          role-duration-seconds: 3600
          role-skip-session-tagging: true

      - name: Test API
        env:
          COGNITO_URL: ${{ secrets.COGNITO_URL }}
          CLIENT_ID: ${{ secrets.COGNITO_CLIENT_ID }}
          USERNAME: ${{ secrets.COGNITO_USERNAME }}
          PASSWORD: ${{ secrets.COGNITO_PASSWORD }}
        run: |
          npm ci

          node ./portman/get-auth-token/user-password-auth.mjs
          npx @apideck/portman --cliOptionsFile portman/portman-cli.json --baseUrl ${{ inputs.BASE_URL }}
```

There are two considerations to take into account here. If you are going to have a user per repository it might become a burden to maintain these users as your applications and repositories grow. On the other hand, if you share a single user with all your repos you might be introducing security vulnerabilities since it will be easier to have this password compromised. 

## Wrap Up
In this post we were able to update our authentication to use the user-password flow instead of M2M for our APIs, this allows us to stay within the Cognito free tier. We were able to verify that we can still authenticate with Postman and our test automations. Hopefully this shows the flexibility there is with Cognito and how you can configure it differently to satisfy your use cases. I will be exploring the other authentication flows so you can easily choose the one that makes more sense for you.
