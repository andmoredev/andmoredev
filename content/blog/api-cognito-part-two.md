+++
title = "Secure API Gateway with Amazon Cognito using SAM - PKCE Edition"
date = 2024-05-15T00:00:00-00:00
draft = false
description = "Last week I released a post on how to secure APIs using machine-to-machine authentication. Exactly one day after that AWS Cognito changed their prices where my proposed setup would actually incur cost. In this post I will go through a different setup using the user password auth flow. This will still allow us to authenticate from automations and form Postman while still keeping us in the free tier."
tags = ["AWS", "Security", "SAM"]
[[images]]
  src = "img/api-cognito-pkce/title.png"
  alt = ""
  stretch = "stretchH"
+++

On my post called [Secure API Gateway with Amazon Cognito using SAM](https://www.andmore.dev/blog/api-cognito/) I talked about different Auth terms and walked through a setup to use the [Client Credentials Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow), but Cognito has recently introduced [pricing changes for machine-to-machine authentication](https://aws.amazon.com/about-aws/whats-new/2024/05/amazon-cognito-tiered-pricing-m2m-usage/) that will make this cost us and my main goal is to do this while staying in the free tier for personal projects that will not be generating any income. That is why in this post I am going to setup Amazon Cognito using a different flow called user password-based authentication. With this type of authentication we are charged based on the Monthly Active Users (MAUs) and AWS gives you the first 50,000 MAUs for free, and in my case this will usually stay at 1 per project, so I should be fine.

## How does the user password flow work?
Initially I thought I could use the [Basic Auth](https://en.wikipedia.org/wiki/Basic_access_authentication) where you provide the encoded username and password in the *Authorization* header but that is now how this works. In the image below I have all the interactions that happen to get an authenticated request.

![Authentication flow interactions](/img/api-cognito-pkce/USER_PASSWORD_AUTH-flow-summary.png)

Let's dive deeper into each interaction
1. Caller makes a request with the username and password to your API.
1. API makes a call to 
1. User provides the username and password. This will initiate the authentication with Cognito which will return the access, id and refresh tokens. These are returned to the user.

1. When making API calls the user will provide the *id token* in the headers, the API is now responsible to verify this token against Cognito to make sure a valid token is being provided.



## Setting it all up with SAM
We are going to stick with a similar architecture for the Amazon Cognito resources to simplify each APIs configuration and to be able to use the same user directory for all our APIs and not have to setup our user for each stack we create.

![CloudFormation Stack Architecture](/img/api-cognito-pkce/stack-architecture.png)

Now let's see how each of these pieces are set up using SAM.

### Auth Stack
This stack has the resources that will be consumed by our API stacks.
Full auth stack setup can be found [in this GitHub repository](https://github.com/andmoredev/cognito-auth)

#### 1. Updates to User Pool
Unlike our M2M setup that didn't need any properties for the user pool, for this type of authentication we need to setup properties that are specifying the attributes to be hosted for the user such as email, names, etc.
```diff yaml
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
* UsernameAttributes - this is specifying what you allow as a user name. The options here are *email* or *phone_number*
Now we can go and add authentication to our API stack using this user pool.
* AutoVerifiedAttributes - attributes tha you allow to be verified automatically by cognito. For example we are using *email* this means Cognito will automatically send a verification email to verify the user. This requires the next attributes setup to work though. If we didn't set this attribute an administrator would have to manually verify users in Cognito.
* VerificationMessageTemplate - This is where you can set your own template for the email that will get sent to users to verify. For this example we are using a default option provided by Cognito where they will confirm by clicking a link. The other option is *CONFIRM_WITH_CODE* where a user will receive a code and they will have to enter it manually to verify.
* EmailConfiguration - This property allows us to setup the configuration for the sender email for the verification or any other communications happening from Cognito. In our case we are using *COGNITO_DEFAULT* which reduces the amount of setup we need to get custom emails setup. The *COGNITO_DEFAULT* has some limits that you will need to consider if you are using, but right now it works for our simple use case.

### API Stack
Our API Stack is actually simplified when using this flow. Why? We do not need to create a resource server to be able to use an authorization scope since we will not be using OAuth capabilities. 

#### 1. Remove unnecessary resources
We will get rid of our Resource Server to use the default scopes.

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

I few changes need to be done in order for this OAuth flow to work. 
* GenerateSecret - We first get rid of this property since it is not needed for this flow.
* AllowedOauthFlows - When using the User/Password auth flow we do not need to set this property.
* AllowedOAuthScopes - Since we are not using an OAuth flow we do not need ot use scopes.
* SupportedIdentityProviders - We are going to use *COGNITO* as our only provider. You can configure different identity providers to simplify your users sign in by using Google, Facebook or any of the supported providers.
* ExplicitAuthFlows: this is where we will configure the *ALLOW_USER_PASSWORD_AUTH* which will allow us to authenticate using a username and a password. I've added *ALLOW_REFRESH_TOKEN_AUTH* since it's required but we will not be going to do token refreshes in this post.

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

THAT'S IT!! We have now successfully setup everything needed to be able authenticate our API using the user-password flow.

## Testing it with Postman

### 1. Grab Client Id
We only need the client id when authenticating with the PKCE flow, this is because we are actually going to enter a username and password to authenticate and that is where the token will be coming from. So we will get this value from the console by going into our new application client.

![AWS Console showing an application client in Cognito where we can grab the client id](/img/api-cognito-pkce/cognito-client-keys.png)

### 2. Setup request authentication
Let's setup the properties for the authentication in Postman.  
* Grant type - This value needs to be *Authorization Code (with PKCE)*.
* Access Token URL - This is the same as what we had for our CCF flow. It should look something like this `https://[your-domain].auth.us-east-1.amazoncognito.com/oauth2/token`
* Client ID - Put the value that you got from the console.
* Scope - Using the default values provided by Cognito: `email openid profile`. Unless your application explicitly needs more scopes you really only need the *openid* scope to successfully authenticate.

![Postman Authorization Configuration with all the values mentioned above filled in](/img/api-cognito-pkce/postman-authorization-configuration.png)

### 5. Get the JWT
We can now get a new access token by going to the bottom of the Authorization section and pressing the *Get New Access Token* button. This will open up a tab for you to sign in/sign up.

![Cognito Sign Up page](/img/api-cognito-pkce/cognito-signup.png)

Once you've entered the email and the password you want for that user you will click the *Sign up* button. This will send an email to the one provided and you will need to click the provided link to verify the user. Once the user is verified you can click the *Continue* button and will be prompted to sign in with the information you entered. If the sign in is successful you will be brought back to Postman (remember the Callback URL that is why you are returned). with that you can then proceed and use the token provided for your request

![Postman showing a successful request](/img/api-cognito-pkce/postman-success.png)

## Wrap Up
In this post we were able to update our authentication to use the PKCE flow instead of M2M for our APIs, this allows us to stay within the Cognito free tier and will not have any cost. We were able to verify that I can still use Postman to authenticate my requests with this flow. By doing this change we now have users in our User Pool that we can manage separately, this also allows us to use this same pattern when implementing user interface. Hopefully this shows the flexibility there is with Cognito and how you can configure it differently to satisfy your use cases.
