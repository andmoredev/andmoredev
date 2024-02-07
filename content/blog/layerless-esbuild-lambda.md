+++
title = "Drop the layers, bundle up with ESBuild instead"
date = 2024-02-07T00:00:00-00:00
draft = true
description = "Learn how you can structure your serverless projects to share code between Lambda Functions using ESBuild bundles instead of Lambda Layers."
tags = ["AWS", "Serverless", "Lambda"]
[[images]]
  src = "img/layerless-esbuild-lambda/title.png"
  alt = "Image of a person removing a jacket into a stack of other jackets the person has already removed"
  stretch = "stretchH"
+++

I've seen a lot of posts around the problems that Lambda Layers bring, a very good one is called [You shouldn't use Lambda Layers](https://aaronstuyvenberg.com/posts/why-you-should-not-use-lambda-layers) by [AJ Stuyvenberg](https://twitter.com/astuyve). In this post AJ explains the myths and cons of using Lambda Layers. What is not easy to find is clear examples on how to actually get rid of Lambda Layers by using a bundler. In this post we will go through a structure and configuration that allows us to remove Layers by using ESBuild to bundle the dependencies and shared code for your functions.

## Backstory
I've been using Lambda Layers for a while as a way to share dependencies and code between different Lambda functions. This was done as a way to keep the code editable from the AWS console and reduce the amount of times we include an npm dependency in the package.json of functions. We had other problems than what AJ mentioned in his post, we noticed that once the project got to a certain size, deployments are a pain. Every deployment creates a new version of the layer which then creates an update for every single Lambda function that's consuming the layer. And this is why I started looking into moving away from layers and instead bundling with ESBuild.

[Link to complete example on GitHub](https://github.com/andmoredev/layerless-esbuild-lambda)

## Configuring ESBuild for Lambda Functions in SAM
The first thing we need to do, is to add the configuraton to let SAM know that we want to build our function using ESBuild. This is done using the 'Metadata' attributes as seen below.
```yaml
Metadata:
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    OutExtension:
      - .js=.mjs
    Target: es2020
    Minify: false
    Sourcemap: true
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);
    External:
      - '@aws-sdk/client-secrets-manager'
```

Let's break these down:
* **BuildMethod** - This is where we tell it to use `esbuild` to bundle the Lambda function code.
* **Format** - Since we are working with ECMAScript Modules we will want our format to be `esm`.
* **OutExtension** - The artifact that gets outputted by `esbuild` has a `.js` extension, since we have specified `esm` for the format, we need to output as `.mjs` which is how NodeJS knows that it is working with ESM.
* **Target** - Used to specify the ECMAScript version to use. Here we will be using the default value of `es2020`.
* **Minify** - this can help reduce the bundle size even more but it makes the code unreadable. For my example I will keep it as false so I can still read the code of my deployed function in the AWS console.
* **Sourcemap** - Attribute to specify if you want to include the sourcemap in your function. Including this allows for a better troubleshooting experience. This is because you will be able to get the stack trace and line numbers from the original code and not the bundled, the downside is that this may have performance impact on your function since it makes it bigger.
* **EntryPoints** - specify the entry points for the application.
* **Banner** - this is not documented by AWS. This property allows us to workaround an issue that ESBuild has with depenedencies that are using `require`. [Link to GitHub issue with the solution](https://github.com/aws/aws-sam-cli/issues/4827#issuecomment-1574080427).
* **External** - Used to exclude specific packages from the bundle. In this case we are importing the `getSecret` action from the `@aws-lambda-powertools/parameters/secrets` package. This package requires the `@aws-sdk/client-secrets-manager` as a dependency, since we know this is already included in the NodeJS 20.X runtime, we exclude it so we use the one included instead of us adding it.

> If you have a dependency that needs to be excluded for all functions my recommendation would be to put that in the devDependencies instead.

When we run `sam build` ESBuild will take care of bundling the shared modules that are being imported in the function from the `src/shared` directory. It will also look at any other dependencies and include them from the `node_modules`.

## Dependency management improvements
There are many ways to structure your project, the one shown in my example is by far the simplest one I could find to reduce the effort of managing npm dependency versions.
How am I doing this? I have a single package.json that includes all the dev and non dev dependencies, when bundling with ESBuild it will take care of only including the dependencies needed and ignore the rest. By keeping a single package.json, we can reduce the amount of pull requests that a tool like Dependabot would open saving a lot of hours of reviewing, testing and merging.

## Configuration for Common JS
The use of ESBuild is not restricted to ESM only, if you are using Common JS the configuration is very similar, we need to exclude a few of the attributes. Below is an example for the same function in Common JS

```yaml
Metadata:
  BuildMethod: esbuild
  BuildProperties:
    Minify: false
    Target: es2020
    Sourcemap: true
    EntryPoints:
      - index.js
    External:
      - '@aws-sdk/client-secrets-manager'
```

> If you are interested in migrating from Common JS to ESM, I wrote an article on [How to update Lambda functions from Common JS to ECMAScript](https://www.andmore.dev/blog/update-commonjs-to-esm/)

## Wrap Up
We learned how you can configure ESBuild to bunlde your functions code instead of using Layers. We were also able to keep a single source for the npm dependencies of our project which reduces the ongoing maintenance. I hope this helps you easily get setup with all your future projects.