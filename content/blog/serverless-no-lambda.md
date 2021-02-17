+++
title = "How to build a Serverless API in AWS without using a single lambda"
date = 2020-03-26T15:56:00-06:00
draft = false
tags = ["aws", "serverless", "lambda", "dynamodb"]
description = "Learn what is needed to build a Lambdaless API using AWS API Gateway, DynamoDB, OpenAPI and CloudFormation."
[[images]]
  src = "img/2020/03/pic01.jpg"
  alt = "Computer Screens"
  stretch = "stretchH"
+++

A common way people build serverless APIs is by routing an API Gateway request to an AWS Lambda. This will make another request to a different AWS Service. It often goes unnoticed that API Gateway can integrate with other AWS Services without the need of Lambda.

Most of the operations that we do in Lambda to fulfill a request are:

* Gather the information from the input body.
* Map the input from one service to another.
* Map the service's output to what we want to return to the client.

You will see time and time again that you are doing these 3 steps over and over again.

This process is possible without using a Lambda to handle the mappings. API Gateway lets you use [Apache's Velocity Templating Language (VTL)](https://velocity.apache.org/) to do what the Lambda did. This will reduce latency and cost by removing the Lambda invocations from each call.

This article will show you what you need to be able to do this without going into the AWS Console. By avoiding the AWS Console we make sure that the code is ready to be integrated into a CI/CD pipeline.

---

# Let's get started
[Link to example](https://github.com/anmoreno/lambdaless-api/)

The example shows how to create a Products Service that will Create, Read, Update and Delete (CRUD) products from the database.

# File Structure
The solution only requires two files:

*template.yaml* - Defines the DynamoDB table, API Gateway and the role that API Gateway will use to execute. CloudFormation uses this file to deploy all our resources.

*products-openapi.yaml* - this is where all the magic happens. This file has the API definition and the mappings needed for the input and output by using VTL.

# Define resources using SAM template
I am using the [Serverless Appliction Model(SAM)](https://aws.amazon.com/serverless/sam/). SAM is a tool that lets you define your Infrastructure as Code. SAM reads from the *template.yaml* file and runs it through CloudFormation. CloudFormation then creates all the resources in the AWS Cloud.
The resources needed for the service to work are the following:

* **DynamoDB table** - Defines a single table with a [Partition Key](https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/) named ***pk***. The value of this attribute will be a *GUID*. (For more complex data models I would recommend watching [this video](https://www.youtube.com/watch?v=6yqfmXiZTlM) by Rick Houlihan)

```yaml
  ProductsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: products
      KeySchema:
        - AttributeName: pk
          KeyType: HASH
      AttributeDefinitions:
        - AttributeName: pk
          AttributeType: S
      BillingMode: PAY_PER_REQUEST
```


* **API Gateway** - Loads the API definition from the OAS file. It is using a Transform to be able to use [intrinsic functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) to get things like ARNs, Parameters, etc.
```yaml
  ProductsAPI:
    Type: AWS::Serverless::Api
    Properties:
        StageName: Lambdaless
        DefinitionBody:
          'Fn::Transform':
            Name: AWS::Include
            Parameters:
              Location: 's3://[YOUR_BUCKET_NAME]/products-api-open-api-definition.yaml'
```

* **Role** - Role that gives API Gateway the permissions needed to be able to run DynamoDB actions.
```yaml
  ProductsAPIRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - apigateway.amazonaws.com
            Action:
              - sts:AssumeRole
      Policies:
        -
          PolicyName: ProductsAPIPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                - "dynamodb:PutItem"
                - "dynamodb:UpdateItem"
                - "dynamodb:DeleteItem"
                - "dynamodb:GetItem"
                - "dynamodb:Scan"

                Resource: !GetAtt ProductsTable.Arn
```


# Create API Gateway endpoints using Open API
A simple way to define your API endpoints is by using the [Open API Specification (OAS)](https://www.openapis.org/). OAS has become the industry standard for API definitions. There are a lot of tools that read the file and generate documentation. [Postman](https://www.postman.com/) and [SwaggerHub](https://swagger.io/tools/swaggerhub/) are some of the applications that do this.

CloudFormation will create the API by using what you already defined using OpenAPI. You have to use Amazon's [custom OAS extensions](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-api-gateway-extensions.html), to be able to set your VTL I/O mappings.

[This Open API Specification example](https://github.com/anmoreno/lambdaless-api/blob/master/products-openapi.yaml) shows the definition of 5 RESTful endpoints. I will be highlighting several parts of this file to better understand what is happening.
First you have the paths, each path will then have each of the operations that it will run: GET, POST, PUT, DELETE.
In the example API we have two paths:

**/products**
This defines two endpoints:
* GET - Get list of all the products.
* POST - Create new Product.

**/products/{productId}**
This path takes in a *productId* to work on a specific product. The endpoints that we use this path for are:
* GET - Get products details.
* PUT - Update product item with the input information.
* DELETE - Delete product.

There are three sections under each of the endpoints:
* requestBody - this section only applies to endpoints that take in an input (POST and PUT in the example). It defines the expected input model. API Gateway validates that the input has expected model. If the validations fail it will return a 400 Bad Request.
* responses - list of the expected responses when calling the endpoint.
* x-amazon-apigateway-integration - This is a custom AWS extension. It defines the mappings for the inputs and outputs for the call to DynamoDB. We will go deeper into this section below.

# Breaking down x-amazon-apigateway-integration extension
The example below shows how the endpoint that adds products implements this custom extension.

```yaml
x-amazon-apigateway-integration:
        httpMethod: POST
        type: AWS
        uri: { "Fn::Sub": "arn:aws:apigateway:${AWS::Region}:dynamodb:action/PutItem" }
        credentials: { "Fn::GetAtt": [ ProductsAPIRole, Arn ] }
        requestTemplates:
          application/json:
            Fn::Sub:
              - |-
                {
                  "TableName": "${tableName}",
                  "Item": {
                    "pk": {
                      "S": "$context.requestId"
                    },
                    "productname": {
                      "S": "$input.path("$.name")"
                    }
                  }
                }
              - { tableName: { Ref: ProductsTable } }
        responses:
          default:
            statusCode: 201
            responseTemplates:
              application/json: '#set($inputRoot = $input.path("$"))
                {
                    "id": "$context.requestId"
                }'
```

There are a lot more properties that can be set for this integration, for a full list go [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration.html). I will be highlighting a few that are being used in the example:

* **uri** - Endpoint that API Gateway will be calling to execute our action. In the example it's calling the [DynamoDB Put Item](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html) endpoint.

* **credentials** - It uses the role that was defined in the * template.yaml. It's using the *[GetAtt](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getatt.html)* intrinsic function to get the Roles ARN.

* **[requestTemplates](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration-requestTemplates.html)** - This takes the body of the request and maps it to the DynamoDB input using VTL. The first item of the array is using the *requestId* to set the *ProductId*. To see a list of all the variables available in the *$context* [go here](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#context-variable-reference). This also makes use of the *[$input](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#input-variable-reference)* variable to get the values from the input, the example is using it to get the *name* attribute. In the second item we use the *[Fn::Sub](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html)* intrinsic function to get the Table Name defined in the *template.yaml*.

* **[responses](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration-responses.html)** - Contains the responseTemplates. This is using the same *$input* and *$context* variables to build the response mapping.
In the next example you can see how to use a *foreach* expression using VTL. This mapping is creating a products array as the response of the API endpoint.
```yaml
responseTemplates:
              application/json: '#set($inputRoot = $input.path("$"))
                          {
                            "products": [
                              #foreach($elem in $inputRoot.Items) {
                                "id": "$elem.pk.S",
                                "name": "$elem.productname.S",
                              }#if($foreach.hasNext),#end
                              #end
                            ]
                          }'
```

There are a lot more things that can be done with Open API and Amazon API Gateway Extensions. To get more information on how you can use these go to these links:
* [Open API documentation](https://swagger.io/docs/specification/about/)
* [Amazon API Gateway Open API Extensions](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions.html)

# Deploy the Service to AWS
As I mentioned before this is all done without touching the AWS console. To get the application deployed to AWS there are a few things that need to happen:

1\. The latest version of the Open API Specification file needs to be copied into an S3 bucket.
```bash
aws s3api create-bucket --bucket [BUCKET-NAME]
aws s3 cp ./products-openapi.yaml s3://[BUCKET-NAME]/
```

The first command will create the S3 bucket where the Open API
file will live. The second command copies the file into the S3
bucket.

> *[BUCKET-NAME] should be replaced with a unique bucket name.
> Bucket names are global, if someones account already has a
> bucket named the same way this command will fail.*

2\. SAM needs to build the artifacts that are going to be deployed. This is done by running the following SAM command:
```bash
sam build
```
This creates a folder named .aws-sam that contains the built
artifacts.

3\. With the artifacts built SAM can deploy them to the cloud.

> *As of version v0.33.1 the sam cli introduced the capability to
> deploy using a samconfig.toml file. You can have the sam cli
> generate this file by running sam deploy in guided mode.*

To create the samconfig.toml file you need to run the following command:
```bash
sam deploy --guided
```
This will ask you a few simple questions to be able to populate
the config file. Everything will be available to test once the
script is done.

# How to test the API
To test this we need the deployed APIs URL. The easiest way to get the URL is by going into API Gateway in the AWS Console. But I promised that we would do everything without going into the console. The solution to this is by using the [aws cli](https://aws.amazon.com/cli/).
For security reasons the APIs URL is not accessible by calling an endpoint, which means we need to build it by hand.
The Cloudformation Stack metadata contains one of the pieces needed to build the URL. The metadata is retrieved by executing the following command:
```bash
aws cloudformation describe-stack-resources --stack-name products-service --logical-resource-id ProductsAPI
```
You can find the *stack-name* in *samconfig.toml* file. The *logical-resource-id* is the name that you used for our API in the *template.yaml*.
The result of the command should look like this:
```json
{
    "StackResources": [
        {
            "StackName": "products-service",
            "StackId": "arn:aws:cloudformation:us-east-1:1234567890:stack/products-service/12345678-9a01-23bc-efgh-1j3kl984m932",
            "LogicalResourceId": "ProductsAPI",
            "PhysicalResourceId": "mx266abc0g",
            "ResourceType": "AWS::ApiGateway::RestApi",
            "Timestamp": "2020-03-24T02:31:27.542Z",
            "ResourceStatus": "UPDATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```
We need three pieces of information to build the URL:
* *PhysicalResourceId* found in the stack metadata.
* *Region* found in the *samconfig.toml file*.
* *Stage* found in the *ProductsAPI* resource in the *template.yaml*.

```
https://[PhysicalResourceId].execute-api.[Region].amazonaws.com/[Stage]
```

In the example it looks like this:
```
https://mx266abc0g.execute-api.us-east-1.amazonaws.com/Lambdaless
```

You can use a tool like Postman to make requests to your API. All you have to do is append the path as defined in the OAS file as well as the necessary *parameters* and *body*.
I included a [Postman collection](https://github.com/anmoreno/lambdaless-api/blob/master/Products.postman_collection.json) in the example that can be imported to view how the requests look.
# Bonus
To simplify deployments I included a *package.json* file. The file has an *npm script* that will take care of executing the deployment commands.
The script looks like this:
```bash
"deploy": "aws s3api create-bucket - bucket [BUCKET-NAME] && aws s3 cp ./products-api-open-api-definition.yaml s3://[BUCKET-NAME]/ && sam build && sam deploy"
```
Now you can use npm run to execute the deployment:
```
npm run deploy
```
By now you should understand how to build and deploy a Lambdaless REST API. As a bonus you get [self documenting APIs by using OAS with Postman](https://learning.postman.com/docs/postman/api-documentation/documenting-your-api/).
I hope you have enjoyed this. You can find a link to the full example [here](https://github.com/anmoreno/lambdaless-api/)