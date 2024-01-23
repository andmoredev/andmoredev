+++
title = "Call third-party endpoints from Step Functions with SAM"
date = 2024-01-01T00:00:00-00:00
draft = true
description = ""
tags = ["AWS", "Serverless"]
[[images]]
  src = "img/layerless-esbuild-lambda/title.png"
  alt = "Image of a person removing a jacket into a stack of other jackets the person has already removed"
  stretch = "stretchH"
+++

# Call third-party endpoints from Step Functions with SAM

[Benoit Boure](https://twitter.com/Benoit_Boure) wrote an article last week on [Calling External Endpoints With Step Functions and the CDK](https://benoitboure.com/calling-external-endpoints-with-step-functions-and-the-cdk), but if you know me I love SAM and wanted to show the same setup but instead of using CDK we will be using SAM.

I will not go into the details of what it does since Benoit did a great job of explaining that in the post I mentioned above, we will simply be going through the things needed to successfully add this integration to your Step Functions.

For this example we will be using the [OpenWeather API](https://openweathermap.org/api) to check the temperature of the provided location and based on the response it will tell us if we need to wear a warm jacket, light jacket or no jacket.

Below is the state machine workflow diagram that we will be executing.  
![Step function diagram](/img/http-invoke-with-sam/stepfunction-diagram.png)

## 1. Create EventBridge Connection
The HTTP task uses an EventBridge Connection to authenticate the request. For the OpenWeather API we need to provide the API Key as a query parameter and not as an Authorization header as you would typically see on other APIs. To accomplish this we add the InvocationHttParameters where we tell it to add the query string parameter of *appid* to the request.

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
There are several permissions needed to successfully execute an endpoint using the HTTP task.
1. Access to the secret that the EventBridge Connection created in Secrets Manager.
2. Permission to retrieve the connection credentials from the EventBridge Connection.
3. Permission to execute the InvokeHTTPEndpoint, in this case the resource needs to be the ARN of the state machine that is executing it. To avoid a cyclical dependency in SAM we are manually creating the ARN, we are adding a wildcard at the end to account for the unique identifier that AWS adds to the resource name.  We are also restricting the HTTP Invocation to only be allowed for a GET to the OpenWeather API Base URL.

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
## 3. Calling the HTTP Invoke State
With all that in place we can now add the step to our state machine as shown below.  
We are specifying the GET method, the connection ARN to our EventBridge Connection that we created, pointing it to the correct OpenWeather URL and then providing the latitude and longitude of the location we would like to get the temperature from.
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

You can find the complete example on [this github repository](https://github.com/andmoredev/http-invoke-with-sam).