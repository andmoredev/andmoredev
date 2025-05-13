+++
title = "Enhancing AppSync Event APIs with Data Sources and ... Lambda Powertools? "
date = 2025-05-03T00:00:00-00:00
draft = false
description = "You heard that right, AWS AppSync released a feature to include data sources for your event APIs, with one of them being Lambda. And Lambda powertools added a new event handler feature that simplifies the processing of events. Follow along to get AppSync Events set up using SAM, explore data sources and get it all working. "
tags = ["AWS", "Serverless", "AppSync", "Powertools"]
[[images]]
  src = "img/log-buffering/title.jpg"
  alt = " "
  stretch = "stretchH"
+++

With the recent release of data source integrations for AppSync Events you can now integrate with services like DynamoDB or Aurora. But the bigger one is the integration with Lambda functions as this opens up the door to anything else you might need. This closes one of the gaps that this service had, which was event storage. The setup was not as straightforward and easy as other things in AWS, so hopefully this post helps you get a jump start in setting this up. 

In this post we will be going through the following:

* Key Concepts
* Setting up AppSync Events API with SAM
* Using a DynamoDB data source with code for subscriptions
* Triggering a Lambda function for publishing events.
* Simplifying everything with AWS Lambda Powertools

### Key Concepts

#### AWS AppSync Overview
AWS AppSync Events gives your applications the ability to do real-time communication with serverless Websocket APIs. With this service your applications can publish messages to millions of subscribers without any code. It takes care of managing the connections and scaling to meet your applications needs.  
If you want to take a deep dive into how they work I highly recommend the post [AWS AppSync Events - Serverless WebSockets Done Right or Just Different?](https://www.ranthebuilder.cloud/post/appsync-events-websockets). 

#### Channel namespaces
This is what is used to expose the channels available on your Event API. The cool thing is that a channel is made up of multiple segments. Meaning you can have a channel named `/andmoredev/public/chat/support`, where the beginning of the segment (andmoredev), is the namespace. Why is this useful? The namespace contains the configuration, meaning you could have thousands of channels under the namespace that share the same configuration (e.g. authentication). 
A big problem with this is that anybody can subscribe or publish messages to any channel as long as the namespace exists, to help with this, and add more logic to the processing of these events we have event handlers.

#### Event handlers
These are snippets of code that run on a special Javascript runtime called APPSYNC_JS. Handlers intercept messages before they are sent back to the subscribers and allow you to run custom business logic to filter or transform events. That is all that you could do with handlers until the recent introduction of data sources into AppSync events.

#### Data sources
These are resources that AWS AppSync Events can interact with from the event handlers allowing you to properly process and route events. At the time of writing the supported data sources are

* Amazon Bedrock runtime
* Amazon DynamoDB
* Amazon EventBridge
* Amazon OpenSearch Service
* AWS Lambda
* HTTP endpoint
* Relational Database (RDS)

One of the biggest gaps that this service when it was released was the ability to store the events that were getting published. Imagine having a live chat where you can't view the history later on, that wouldn't be a great experience. With this new feature you can automatically store events using event handlers and data sources with very minimal code. This also allows you to do any type of custom processing, and with Lambda being a supported resource, the sky is the limit.

To be hones they feel more like targets or destinations rather than sources since you are actually sending data to these locations to be stored or processed.

### Configuration options
### Let's see this in action!
While there is not a ton of configuration it was not as straightforward to setup using SAM. It's probably since it's very new and the documentation hasn't propagated all of these changes yet. Here I'll be talking about all of the configuration settings needed to setup AppSync Events, Data Sources and their Event Handlers.

#### Set up appsync API with SAM
The first thing to do is to create the API resource

```yaml
  AppSyncEventApi:
    Type: AWS::AppSync::Api
    Properties:
      EventConfig:
        AuthProviders:
          - AuthType: API_KEY
        ConnectionAuthModes:
          - AuthType: API_KEY
        DefaultPublishAuthModes:
          - AuthType: API_KEY
        DefaultSubscribeAuthModes:
          - AuthType: API_KEY
```
This creates the AppSync API and sets up the Auth Providers allowed, and sets the default auth mode for creating a connection, for subscribing to a channel and publishing to a channel.

```yaml
  AppSyncEventApiKey:
    Type: AWS::AppSync::ApiKey
    Properties:
      ApiId: !GetAtt AppSyncEventApi.ApiId
```

The resource above will create the API Key that will be used to be able to connect, subscribe and publish messages.

The other accepted authentication modes allowed are:
* AMAZON_COGNITO_USER_POOLS
* AWS_IAM
* OPENID_CONNECT
* AWS_LAMBDA

If we wanted to also allow authentication using my existing Cognito setup explained in my post [Using Amazon Cognito With The User-Password Flow](https://www.andmore.dev/blog/api-cognito-user-password/) we can update our template to include the following parameters to include the information from my existing Cognito User Pool.

```yaml
Parameters:
  CognitoUserPoolId:
    Type: 'AWS::SSM::Parameter::Value<String>'
    Default: '/andmoredev-auth/CognitoUserPoolId'

  CognitoUserPoolArn:
    Type: 'AWS::SSM::Parameter::Value<String>'
    Default: '/andmoredev-auth/CognitoUserPoolArn'

  CognitoUserPoolUrl:
    Type: 'AWS::SSM::Parameter::Value<String>'
    Default: '/andmoredev-auth/CognitoUserPoolUrl'
```

And create a new user pool client to be used by this application
```yaml
  CognitoUserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      UserPoolId: !Ref CognitoUserPoolId
      SupportedIdentityProviders:
        - COGNITO
      ExplicitAuthFlows:
        - ALLOW_USER_PASSWORD_AUTH
        - ALLOW_REFRESH_TOKEN_AUTH
```

To include this as an auth provider and use it as a default auth type as well we need to update our event API as follows:

```yaml
  AppSyncEventApi:
    Type: AWS::AppSync::Api
    Properties:
      Name: EventApi
      EventConfig:
        AuthProviders:
          - AuthType: API_KEY
          - AuthType: AMAZON_COGNITO_USER_POOLS
            CognitoConfig:
              UserPoolId: !Ref CognitoUserPoolId
              AwsRegion: !Ref AWS::Region
        ConnectionAuthModes:
          - AuthType: API_KEY
          - AuthType: AMAZON_COGNITO_USER_POOLS
        DefaultPublishAuthModes:
          - AuthType: API_KEY
          - AuthType: AMAZON_COGNITO_USER_POOLS
        DefaultSubscribeAuthModes:
          - AuthType: API_KEY
          - AuthType: AMAZON_COGNITO_USER_POOLS
```

This will allow us to use a JWT to authenticate instead of a simple API Key.

Logging
Web ACL


#### Data sources
Now we'll create the data sources to be use by our event handlers. We will be using a DynamoDB data source for the OnSubscribe events, to store the channels. For the OnPublish event we will use a Lambda data source to process all the messages that are being sent.

We first need to define our DynamoDB table so we can reference it in our data source.
```yaml
  DynamoDBTableName:
    Type: AWS::DynamoDB::Table
    Properties:
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
```

The data source needs to give AppSync permissions to be able to do actions on it. In this case we will be giving it permissions to do a putItem on our table.
```yaml
  AppSyncDynamoDBDataSourceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - appsync.amazonaws.com
            Action:
              - sts:AssumeRole
      Policies:
        - PolicyName: AppSyncDynamoDBDataSourcePolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:PutItem
                Resource:
                  - !GetAtt DynamoDBTableName.Arn
``` 

We now have everything ready de add our data source.
```yaml
  AppSyncDynamoDBDataSource:
    Type: AWS::AppSync::DataSource
    Properties:
      ApiId: !GetAtt AppSyncEventApi.ApiId
      Name: DynamoDBDataSource
      Type: AMAZON_DYNAMODB
      ServiceRoleArn: !GetAtt AppSyncDynamoDBDataSourceRole.Arn
      DynamoDBConfig:
        TableName: !Ref DynamoDBTableName
        AwsRegion: !Ref AWS::Region
```

This is creating a datasource for our API by giving it the ApiId, setting the type of datasource, assigning it the role we created and adding the type specific config, which in this case is just the TableName and the AwsRegion.

Now we'll create the Lambda datasource, we'll be using the function defined in the project called `AppSyncEventHandler` but this also needs a role to grant AppSync permissions to invoke this function.

```yaml
AppSyncLambdaDataSourceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - appsync.amazonaws.com
            Action:
              - sts:AssumeRole
      Policies:
        - PolicyName: AppSyncLambdaDataSourcePolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - lambda:InvokeFunction
                Resource:
                  - !GetAtt AppSyncEventHandler.Arn
                  - !Sub ${AppSyncEventHandler.Arn}:*
```

And the data source resource should be fairly similar to the DynamoDB one as shown below

```yaml
  AppSyncLambdaDataSource:
    Type: AWS::AppSync::DataSource
    Properties:
      ApiId: !GetAtt AppSyncEventApi.ApiId
      Name: LambdaDataSource
      Description: Testing the Lambda DataSource Type
      Type: AWS_LAMBDA
      ServiceRoleArn: !GetAtt AppSyncLambdaDataSourceRole.Arn
      LambdaConfig:
        LambdaFunctionArn: !GetAtt AppSyncEventHandler.Arn
```

With all of this in place we can now go ahead and create our Namespace with its event handlers.

#### Namespaces
Let's define our namespace now.
```yaml
  AppSyncEventApiChannelNamespace:
    Type: AWS::AppSync::ChannelNamespace
    Properties:
      ApiId: !GetAtt AppSyncEventApi.ApiId
      Name: AndMoreChat
      CodeHandlers: "{{StringCodeHandler}}"
      HandlerConfigs:
        OnSubscribe:
          Behavior: CODE
          Integration:
            DataSourceName: !GetAtt AppSyncDynamoDBDataSource.Name
        OnPublish:
          Behavior: DIRECT
          Integration:
            DataSourceName: !GetAtt AppSyncLambdaDataSource.Name
            LambdaConfig:
              InvokeType: REQUEST_RESPONSE
```
There is quite a bit that I want to unravel here. The two first properties are pretty straightforward to add this namespace to our existing API and the name of the namespace. This is what will be used as the first segment when subscribing or publishing to a channel.

The next one was the trickier one to get working with SAM and CloudFormation. The CodeHandlers attribute is where the AppSyncJS code will be provided which visually lives here.
![Event Handler in the AWS Console](/img/appsync-events-datasources/code-handler.png)

There are two ways to provide this data, CodeHandlers or CodeS3Location. My immediate thought was to use the one in S3, but it has a MAJOR downside. If you only update the code and have that uploaded in S3, if you redeploy there are no changes identified in your stack, therefore nothing gets updated. Which is why I decided to go the CodeHandlers route which is simply the stringified code. But, this also has it's major downsides as this is not easy to maintain AT ALL, which is why I still do all the coding in a `mjs` file and simply stringify the code and replace in the template file using the script found [here](https://github.com/andmoredev/appsync-events/blob/main/scripts/escape-appsync-handler.mjs#L1-L27). This same script can be used in the CI/CD pipelines, the code can be run against unit tests, and SAM identifies changes to be deployed whenever there is a code change.

Now to the most important part! Configuring the event handlers. As mentioned for the OnSubscribe we want to use our DynamoDB data source. Since the interaction with this resource is via the code we need to set the `Behavior` to `CODE`. This will tell AppSync that this will be defined in the code handler, and if it's not it will actually let you know that it's missing. Now let's take a look at the actual code!
```javascript
import * as ddb from '@aws-appsync/utils/dynamodb'
import { util } from '@aws-appsync/utils'

export const onSubscribe = {
  request(event) {
    if (event.info.channel.segments.length > 2) {
      util.error('Invalid Room Name - Too Many Segments');
    }

    if (event.info.channel.segments.includes('*')) {
      util.error('Invalid Room Name - Wildcards');
    }

    if(!event.identity.groups || event.identity.groups?.includes("TestGroup")) {
      util.unauthorized();
    }

    const channel = event.info.channel.path
    const room = event.info.channel.segments[1]
    const timestamp = util.time.nowEpochMilliSeconds();

    return ddb.put({
      key: { id: room },
      item: {
        id: room,
        channel,
        timestamp
      }
    });
  },
  response(ctx) {
    return ctx.result;
  }
}
```

This is using the AppSync utils to make our interactions with DynamoDB easier. Since this is the OnSubscribe event, in this case we are preventing subscriptions to channels that have more than two segments and that have wildcards, we only want to allow direct channel subscriptions. And we've added an extra authorization piece, to make sure that the caller belongs to a specific group in Cognito.
If it is all successful we then store this information in DynamoDB.

For the OnPublish we are using the Lambda function data source. What the Lambda function is not doing much, but it could publish metrics or traces for our observability, filter out offensive words, it can basically do any type of processing you need since you can do anything from a Lambda function, and if you can't, you can trigger the thing that can.

For this data source integration, they've added the `DIRECT` Behavior. Which takes care of invoking the Lambda function without you having to add any code. This is very similar to what we already had with the lambda_proxy in API Gateway. You could still use the `Code` Behavior but you would need to code the lambda invocation yourself. This data source has one extra piece of configuration which is the `InvokeType`. The allowed values are `EVENT`, to invoke the function asynchronously and not wait for the response, and `REQUEST_RESPONSE` to wait for the lambda to do it's processing before responding.


### Wrap up
One of the main points made is that even though this is under the AWS AppSync name it has nothing to do with GraphQL... I know, confusing right?


Until next time!

Andres Moreno
