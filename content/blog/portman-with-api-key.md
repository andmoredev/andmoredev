+++
title = "Add Authentication to Portman API tests"
date = 2022-03-24T16:18:15-06:00
draft = false
description = "Testing secure APIs through automation can be a pain. Thankfully Portman provides an easy way to include authentication in your test automation."
tags = [ "API", "OpenAPI", "Test" ]
[[images]]
  src = "img/portman-with-api-key/cover.jpg"
  alt = "Portman CLI"
  stretch = "stretchH"
+++

When creating an API you want to have security to avoid bad usage and incurring cost by unintended use. Whenever we add a security layer to our APIs any automation that we have around it automatically becomes more complex. Portman provides ways to configure authentication to be used when it runs. In this post we will configuring Portman to successfully run against our secure API.

We will be using the API that we already added Portman configuration to in the [Get better API testing by using Portman post](https://www.andmore.dev/blog/getting-started-portman/). First we need to add security to the API so it only allow access to calls made with a valid API key, after that we can configure Portman to use that API Key and successfully make all the requests needed.

Here is the [source code](https://github.com/andmoredev/lambdaless-api) for the end result. If you want to follow along clone/download [this commit](https://github.com/andmoredev/lambdaless-api/tree/f40b2b19f87f37b3bb6d7624f5f7c44130b0463e).

# Add Security to API Gateway
To add API Key authentication to the API we will be adding the *Auth* property in the API resource in the SAM template as shown below:
```yaml
ProductsAPI:
    Type: AWS::Serverless::Api
    Properties:
        StageName: Lambdaless
        Auth:
          ApiKeyRequired: true
          UsagePlan:
            CreateUsagePlan: PER_API
            UsagePlanName: ProductsAPIUsagePlan
        DefinitionBody:
          'Fn::Transform':
            Name: AWS::Include
            Parameters:
              Location: ./products-openapi.yaml
```

We will also be adding the ApiKeyId in the Outputs section to help us retrieve the API Key later
```yaml
ApiKeyId:
    Description: API Key ApiKeyId
    Value: !Ref ProductsAPIApiKey
```

Once the API is deployed with these changes, it will now require an API Key to be passed in the header. This will cause the Portman tests to fail with a 403 since it is not providing the header yet. I will not go into any more details around how security works for API Gateway since it is not in the scope of this article.

# Get the API Key
There are two ways you can get the API Key that was generated:
* Go to the console and retrieve it from the API Gateway service
* With the CLI. Below I will be explaining how to do it from the CLI.

1. First we need to get the API Id. We can do this by executing the following command:

`aws cloudformation describe-stacks --stack-name products-service`

The response for this command comes back with the Outputs defined in the SAM template where we have included the ApiKeyId. Copy the value presented in the OutputValue.

2. Now we need to run following command replacing the --api-key value with the value copied from the previous step to get the API Key that we can include in our headers:

`aws apigateway get-api-key --api-key REPLACE_WITH_APIID --include-value`

The API Key will be contained in the "value" attribute in the response. Copy this since we will be using it for the portman configuration in a bit.

# Add security definitions in the OpenAPI
If you try to run Portman right now you will get 403 Forbidden responses because Portman is not providing an API key to the request. Portman uses your OpenAPI definition to generate and run the tests, if we do not define the securitySchemes it will not know to apply them. So first we need to setup the securitySchemes supported by our API which in our case is only API Key, to do this we will add the following YAML under the *components* section of our OpenAPI spec.
```yaml
 securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-KEY
```

OpenAPI supports several authenticaton schemes if you wish to learn more about these please look at the [OpenAPI documentation here](https://swagger.io/docs/specification/authentication/).
Portman currently suports the API Key, HTTP basic auth and HTTP bearer token security schemes.

In OpenAPI you can apply a different security scheme per endpoint, in our case we want to apply the same one across the whole API. To do this we will be adding the root level *security* property to define it as shown below.
```yaml
security:
  - ApiKeyAuth: []
```
If we run Portman again we will still get a 403 response since we are still not telling Portman what API Key to use.

# Configure Portman to use an API Key
Portman allows you to configure securityOverwrites in the globals section of your Pormtan config file.
We will be updating ours to include an overwrite for the apiKey.
```json
{
  "securityOverwrites": {
    "apiKey": {
      "value": "INSERT THE API KEY HERE"
    }
  }
}
```

Now if we run Portman we will get a successful run just as we did prior to adding security to the API.

# Recap
In this post we successfully tested a secure API using Portman by
* Adding security to the API
* Updating the OpenAPI spec to define the type of authentication to be used
* Configured Portman to successfully include an API key in the header of every request to be able to automate our API testing for secure APIs

# Resources
* [Source Code](https://github.com/andmoredev/lambdaless-api)
* [Template for adding API Keys to API Gateway](https://davekz.com/aws-sam-api-keys/)
* [OpenAPI security schemes documentation](https://swagger.io/docs/specification/authentication/)
* [Portman documentation](https://github.com/apideck-libraries/portman/blob/main/README.md)