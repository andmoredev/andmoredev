+++
title = "Drop the layers, bundle up with ESBuild instead"
date = 2024-01-01T00:00:00-00:00
draft = true
description = "Find out how you can structure your serverless projects to share code between Lambda Functions using ESBuild bundling instead of Lambda Layers."
tags = ["AWS", "Serverless"]
[[images]]
  src = "img/layerless-esbuild-lambda/title.png"
  alt = "Image of a person removing a jacket into a stack of other jackets the person has already removed"
  stretch = "stretchH"
+++

I've seen a lot of posts around the problems that Lambda Layers bring, a very good one is [You shouldn't use Lambda Layers](https://aaronstuyvenberg.com/posts/why-you-should-not-use-lambda-layers) by [AJ Stuyvenberg](https://twitter.com/astuyve). In this post AJ goes through the myths and the cons of using Lambda Layers. What is not easy to find is a clear example on how to get rid of Lambda Layers in place of shared files using a bundler. In this post we will through the project structure and configuration that allows us to use ESBuild to bundle our Lambda functions to include shared code and dependencies.

* From - to --- basically show my previous project structure and the new project structure and explain the positioning of the package.json and the benefits of that
* SAM config for ESBuild - what each of the pieces mean and why we have them
* Plug the CJS to ESM updates and benefits.



