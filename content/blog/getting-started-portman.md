+++
title = "Get better API testing by using Portman"
date =  2021-10-28T19:41:28-05:00
draft = false
description = "Automate your API tests by using your OpenAPI definitions."
tags = ["AWS", "OpenAPI", "Test"]
[[images]]
  src = "img/getting-started-portman/portmancli.jpg"
  alt = "Portman CLI"
  stretch = "stretchH"
+++

With the rise of spec-driven API development, more tools are being created that allow the use of an OpenAPI definition to design, test and validate your APIs. One tool that was recently brought to my attention is Portman which is an API testing library that *makes it easy to ensure your APIs are reliable while keeping your documentation in sync with the API definition*.

Portman has a CLI that simplifies the setup needed to get up and running. In this post we will be setting up Portman and going through a few simple tests that we want to run against our API.

I will be using the API that was created in [How to build a serverless API in AWS without using a single Lambda](https://www.andmore.dev/blog/build-serverless-api-with-no-lambda/), since that API is created by only using the OpenAPI file. Follow the instructions in the article to get it deployed and take note of the outputted URL when the deployment finishes.

![Products URL Output](/img/getting-started-portman/01.jpg)

[Here's a link to GitHub repo](https://github.com/andmoredev/lambdaless-api)

If you are following along make sure to clone/download [this commit](https://github.com/andmoredev/lambdaless-api/tree/819fb40601cca07ec3157e25c950f3845dac7a9e)

## 1. Install Portman
To be able to use the Portman CLI we first need to install it globally by running

``` npm install -g portman ```

## 2. Initialize Portman
Now that it is installed we can easily use the CLI to get our initial setup ready.

``` portman --init ```

This will prompt us for several things to make sure everything is setup. Below you can see all the selections I made to get it setup.

![Portman CLI Configuration](/img/getting-started-portman/02.jpg)

## 3. Add Default Configuration
In the initialization we selected to use our own custom configuration. To start off we will simply copy the default configuration [from here](https://github.com/apideck-libraries/portman/blob/main/portman-config.default.json) into the portman-config.json file under the portman directory.

The default configuration file simply defines contract tests that will be making sure that all the endpoints return a status of success (2XX), the content type returns as defined in the OpenAPI, the response has a JSON Body and the response body matches whats defined in the OpenAPI

## 4. Run Portman

Now that everything is setup we can simply run Portman with the command specified at the end of the setup.

``` portman --cliOptionsFile portman/portman-cli.json  ```

## 5. Fixing Issues found in the OpenAPI spec

### **Error #1 - Missing URL**
When the previous command finishes running it will uncover the first issue in the OpenAPI. By looking at the error in the terminal, we can see the URL that it is trying to call has nothing but the path defined in the OpenAPI spec.

![Missing Server URL](/img/getting-started-portman/03.jpg)

This is because Portman uses the *[server and base URL](https://swagger.io/docs/specification/api-host-and-base-path/)* section from the OpenAPI to execute the requests. To fix this we are going to go to the products-openapi.yaml file and add the servers section to the file, for the URL I will be using the Products API we already grabbed at the beginning.

The end result should look like this:

![OpenAPI servers](/img/getting-started-portman/04.jpg)

### **Error #2 - Response Schema does not match**

When we rerun portman we get the following errors.

![Portman Failures](/img/getting-started-portman/05.jpg)

What this is telling us, is that the `GET /products` request is not returning in the expected format defined in the OpenAPI.

If we look at our OpenAPI it defines the response schema as an array of products:
``` json
[
  {
  }
]
```

When we compare that with what is being done in the response template, this is returning an object with a *products* array which is something that looks like this:
``` json
{
  "products": [
  ]
}
```

To fix this we are going to remove the products attribute from the response template and simply return the array.
The response template should look like this:

![Portman Failures](/img/getting-started-portman/06.jpg)

To confirm this works we will need to redeploy the service and when it's finished deploying let's run Portman again and we will see that everything runs successfully now.

![Successful Run](/img/getting-started-portman/07.jpg)

## 5. Assigning Variables & Overwrites
To be able to use values from a requests response we can use the assignVariables section in the portman configuration file to specify which endpoints we will want to do that on as well as which variable to store it in.

In our case we want to use the *id* returned from the `POST products` request on all the other requests. To accomplish this our assignVariables section needs to look like below.

``` json
{
  "assignVariables": [
    {
      "openApiOperation": "POST::/products",
      "collectionVariables": [
        {
          "name": "newProductId",
          "responseBodyProp": "id"
        }
      ]
    }
  ]
}
  ```

  Let's go over what this means.

  * openApiOperation - this is telling portman which requests should be adding this variable assigment. The structure of this is *[Method]::[Path]*. The method and the path can be wildcards, if we go back to the default config that we copied it uses *"\*::/\*"*, that is saying *run the tests for every method and every path*.
  * collectionVariables - this is specifying the *name* of the collection variable to be able to reference it in other requests and the *responseBodyProp* defines the path to get the value from the response.

  To use this variable in other requests we are going to update the *overwrites* section. It should look like this:
  ``` json
  {
    "overwrites": [
      {
        "openApiOperation": "*::/products/{productId}",
        "overwriteRequestPathVariables": [
          {
            "key": "productId",
            "value": "{{newProductId}}",
            "overwrite": true
          }
        ]
      }
    ]
  }
  ```

  Let's piece this one out as well

  * openApiOperation - in this situation we are using a wildcard for the HTTP method and specifying exactly which path we want this overwrite to apply to.
  * overwriteRequestPathVariables
    * key - where do you want to replace the value, in our case it is {productId} in the path parameter.
    * value - What is the value you want to replace it with, this can come from a collection variable or simply hardcoded. Here we are using the newProductId specified in the *collectionVariables* section.
    * overwrite - boolean that tells portman if it should overwrite the request path variable.

  When we rerun portman we will see a new comment on the POST request telling us the collection variable was set.

![Collection Variable Setting](/img/getting-started-portman/08.jpg)

And if we look at all the other requests they will be using that same product id, since we specified the overwrite with a wildcard.

![Collection Variable Usage](/img/getting-started-portman/09.jpg)

# Wrap Up
We have learned how simple it is to get contract tests setup using our existing OpenAPI definition. Very quickly you can start finding discrepancies between our API and what we had defined.
These types of tests really help to give users the confidence on our APIs as well as maintaining our documentation up to date easily.
This article only covers the basics but there is a lot more that you can do with Portman to be able to cover most scenarios.
