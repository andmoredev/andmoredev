+++
title = "Using YAML anchors and aliases in a SAM template"
date = 2024-03-26T00:00:00-00:00
draft = false
description = "Learn how you can setup your SAM template to reuse common pieces of config by using anchors and aliases without introducing any problems."
tags = ["AWS", "Serverless", "SAM"]
[[images]]
  src = "img/sam-yaml-anchors/title.png"
  alt = "Image of a diagram in excalibur showing a Step Function calling a Lambda Function and the Lambda Function calling out to the Internet with the Lambda Function crossed out"
  stretch = "stretchH"
+++

Last month I wrote a post about [getting rid of Lambda Layers by using ESBuild](https://www.andmore.dev/blog/layerless-esbuild-lambda/). What I quickly learned is that the *Metadata* attribute has to be copied and pasted for EVERY Lambda function in your stack. I tried using the *Global* section in the SAM template and it turns out it's not supported. I started thinking about how I could reuse the same configuration across my template and found that YAML already has a functionality that does this called [YAML Anchors and Aliases](https://www.educative.io/blog/advanced-yaml-syntax-cheatsheet#anchors). In this post I will go through what YAML aliases and anchors are and how we can use them in SAM.

## What are YAML anchors and aliases?
You can think of *anchors* as assigning a value to a variable to be used in other places. The way you define an *anchor* is by adding an *&* on an entity with the name of the anchor.
```yaml
personName:
  type: object
  properties: &person-name-properties
    firstName:
      type: string
    lastName:
      type: string
```

In the example above we have created an anchor for the *properties* attribute of the *personName* schema with the name *person-name-properties*. Later in the file you can reference the anchor with an alias by using the *\** symbol and the anchor name.
```yaml
person:
  type: object
  properties:
    name:
      type: object
      properties: *person-name-properties
```

When the YAML is proccessed the *person* object will look like this.
```yaml
person:
  type: object
  properties:
    name: 
      type: object
      properties:
        firstName:
          type: string
        lastName:
          type: string
```

YAML allows you to override and/or extend properties to be able to get it into the correct structure. To update the name object to have a *minLength* for the *firstName* and include a new attribute called *middleName* we would do something like this.
```yaml
person:
  type: object
  properties:
    name:
      type: object
      properties: 
        <<: *person-name-properties
        firstName:
          type: string
          minLength: 3
        middleName:
          type: string

```

If you look at the example above we are now using the *<<* attribute which essentially *deconstructs* whats defined in the anchor and allows you to overwrite something by adding the same attribute or add new ones in line. If we process the YAML above we would get the result below.
```yaml
person:
  type: object
  properties:
    name: 
      type: object
      properties:
        firstName:
          type: string
          minLength: 3
        lastName:
          type: string
        middleName:
          type: string
```
With this functionality we can define our base *Metadata* attributes for ESBuild and use them for our template right? .... right?  
![SAM and YAML meme](/img/sam-yaml-anchors/sam-yaml-meme.jpg)

## Why are YAML anchors not as straightforward with SAM?
SAM is strict in what it accepts in the *template.yaml* when doing a deployment, you will not see these errors when doing `sam build` though, so be careful to make sure your template is actually deployable. Here is an example where I'm defining the *esbuild* config metadata at the root of my template and consuming it in a function

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Layerless ESBuild Example
Transform:
  - AWS::Serverless-2016-10-31

esbuild: &esbuild
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    Minify: false
    OutExtension:
      - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);

Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata: *esbuild
```

The built template after running `sam build` looks like this
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Layerless ESBuild Example
Transform:
- AWS::Serverless-2016-10-31
esbuild:
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    Minify: false
    OutExtension:
    - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
    - index.mjs
    Banner:
    - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);
Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: EchoFunction
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Banner:
        - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);
        EntryPoints:
        - index.mjs
        Format: esm
        Minify: false
        OutExtension:
        - .js=.mjs
        Sourcemap: false
        Target: es2020
      SamResourceId: EchoFunction
```
Doesn't look bad right? It looks like the anchor got referenced and placed correctly in my Lambda function.But when we run `sam deploy` we'll get this error.
```bash
Status: FAILED. Reason: Invalid template property or properties [esbuild]
```
It doesn't like the *esbuild* property we've defined since it's not part of the SAM template schema.

## Working around the limitation
To avoid getting an error on deployment we can rename the property to *Metadata* instead. This is an accepted property in SAM that will build and deploy successfully. Below you can see how I renamed *esbuild* to *Metadata*
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Layerless ESBuild Example
Transform:
  - AWS::Serverless-2016-10-31

Metadata: &esbuild
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    Minify: false
    OutExtension:
      - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);

Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata: *esbuild
```
This is great! We can have several Lambda functions that share the same ESBuild configuration. But what happens when we want to define multiple anchors? If we were to add a second *Metadata* attribute we will now have invalid YAML since it doesn't accept duplicate properties names at the same level. The funny thing is this actually builds and deploys correctly so if you have no linting in your processes this might work for you. But there is a better way we can define multiple anchors without sacrificing our linting process.  
Instead of making the *Metadata* property the anchor we can add multple anchors under the *Metadata* property. In the example below I am going to add a second anchor that contains only the *BuildProperties* and I'll update the base ESBuild config to use it.
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Layerless ESBuild Example
Transform:
  - AWS::Serverless-2016-10-31

Metadata:
  esbuild-properties: &esbuild-properties
    Format: esm
    Minify: false
    OutExtension:
      - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);

  esbuild: &esbuild
    BuildMethod: esbuild
    BuildProperties: *esbuild-properties

Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata: *esbuild
```
We've done it! We can now define reusable YAML objects in our SAM templates. If we wanted to extend or overwrite any properties we can use the *<<* attribute.
```yaml
Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        <<: *esbuild-properties
        Minify: true
        External:
          - '@aws-sdk/*'
```

## Wrap up
By using standard YAML functionality and repurposing the SAM *Metadata* property we were able to successfully setup reusable snippets of YAML for our SAM templates. This allows us to greatly reduce the size of the template we are maintaining, it also reduces the risk of errors by giving standard settings to be used throughout the whole file.