+++
title = "Call APIs from Step Functions using SAM"
date = 2024-01-23T00:00:00-00:00
draft = false
description = "Learn what's needed to trigger the HTTP endpoint task state from a Step Function to be able to eliminate the Lambda Functions you have to call APIs"
tags = ["AWS", "Serverless"]
[[images]]
  src = "img/http-invoke-with-sam/title.png"
  alt = "Image of a diagram in excalibur showing a Step Function calling a Lambda Function and the Lambda Function calling out to the Internet with the Lambda Function crossed out"
  stretch = "stretchH"
+++

[Benoit Boure](https://twitter.com/Benoit_Boure) wrote an article last week on [Calling External Endpoints With Step Functions and the CDK](https://benoitboure.com/calling-external-endpoints-with-step-functions-and-the-cdk), but if you know me I love SAM and wanted to show the same setup but instead of using CDK we will be using SAM.

I will not go into the details of how it works since Benoit did a great job of explaining that in the post mentioned above. 

We'll be using the [OpenWeather API](https://openweathermap.org/api) to get the temperature of a place and check if we need to wear a warm jacket, light jacket or no jacket. Below is the state machine workflow diagram that we will be executing.  
![Step function diagram](/img/http-invoke-with-sam/stepfunction-diagram.png)

 [Link to complete example on GitHub](https://github.com/andmoredev/http-invoke-with-sam)

# Steps to follow  
## 1. Create EventBridge Connection
The HTTP task uses an EventBridge Connection to authenticate the request. For the OpenWeather API we need to provide the API Key as a query parameter and not as an Authorization header as you would typically do on other APIs. To accomplish this we add the *InvocationHttParameters* object where we add the query string parameter to the request with the name appid.

```yaml
OpenWeatherConnection:
  Type: AWS::Events::Connection
  Properties:
    Name: 'openweather-connection'
    AuthorizationType: API_KEY
    AuthParameters:
      ApiKeyAuthParameters:
        ApiKeyName: Authorization
        ApiKeyValue: !Ref OpenWeatherAPIKey
      InvocationHttpParameters:
        QueryStringParameters:
          - IsValueSecret: true
            Key: appid
            Value: !Ref OpenWeatherAPIKey
```
## 2. Create Step Function Execution Policy
There are several permissions needed to execute an endpoint using the HTTP task.
1. It needs access to the secret that the EventBridge Connection creates in Secrets Manager.
2. Permission to retrieve the connection credentials from the EventBridge Connection.
3. Permission to execute the InvokeHTTPEndpoint. In this case the resource needs to be the ARN of the state machine that is executing it. To avoid a cyclical dependency in SAM we are parsing the ARN ourselves. I'm adding a wildcard at the end to account for the unique identifier that AWS adds to the end of generated resource names.  We are also restricting the HTTP Invocation to only allow a GET to the OpenWeather API.

```yaml
Policies:
  - Version: 2012-10-17
    Statement:
      - Effect: Allow
        Action:
          - secretsmanager:DescribeSecret
          - secretsmanager:GetSecretValue
        Resource: !GetAtt OpenWeatherConnection.SecretArn
      - Effect: Allow
        Action:
          - events:RetrieveConnectionCredentials
        Resource: !GetAtt OpenWeatherConnection.Arn
      - Effect: Allow
        Action:
          - states:InvokeHTTPEndpoint
        Resource: !Sub arn:aws:states:${AWS::Region}:${AWS::AccountId}:stateMachine:ShouldIWearAJacketStateMachine*
        Condition:
          StringEquals:
            states:HTTPMethod: GET
          StringLike:
            states:HTTPEndpoint: !Ref OpenWeatherBaseUrl
```
## 3. Calling endpoint from the Step Function
With all that in place we can now add the step to our state machine as shown below.  
We are specifying the HTTP method as GET, the connection ARN to our EventBridge Connection as the authentication, OpenWeather base URL and we are appending query parameters to get the temperature of the location and the units we want.
```json
"Get Temperature": {
  "Type": "Task",
  "Resource": "arn:aws:states:::http:invoke",
  "Parameters": {
    "Method": "GET",
    "Authentication": {
      "ConnectionArn": "${OpenWeatherConnectionArn}"
    },
    "ApiEndpoint": "${OpenWeatherBaseUrl}",
    "QueryParameters": {
      "lat.$": "$.latitude",
      "lon.$": "$.longitude",
      "units": "imperial"
    }
  },
  "ResultSelector": {
    "temperature.$": "$.ResponseBody.main.temp"
  }
}
```

# Wrap Up
As you can see it is not that complicated to replace any Lambda Functions you have to call third party endpoints. The permissions can get a little bit complicated but once you undrestand them you can repeat them for other requests you need to do.