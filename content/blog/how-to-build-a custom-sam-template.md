+++
title = "Speed up new serverless application development with SAM customized templates"
date = 2023-02-01T00:00:00Z
draft = true
description = "Learn how you can use custom templates with SAM to speed up initial application setup."
tags = ["AWS", "Serverless", "SAM"]
[[images]]
  src = "img/how-to-build-a-custom-sam-template/sam.jpg"
  alt = "Jenga"
  stretch = "stretchH"
  removeBlur = true
+++

As you start setting patterns and best practices within your projects at some point you will want to share these to simplify other developers lives.  
SAM allows you to initialize projects by using the *init* command and selecting a template for your project. SAM has built-in templates to generate simple Hello-World starter projects, you can find the list of templates [here](https://github.com/aws/aws-sam-cli-app-templates). These will help you get started and get familiar with how to structure your serverless applications, but as your applications grow you will want to add your own flavor to these. This is why SAM provides the ability to use custom templates to startup applications that include your own designs.

# How does `sam init` work?
We first need to understand how the *init* command works.  
`sam init` uses *[cookiecutter](https://www.cookiecutter.io/)* to be able to prompt and resolve all the values for a template.  
You will first have to select if you want to use an existing AWS template or if you want to use a custom template, if you choose the latter you will have to provide the location of the template.   
Once you have a template selected it will use cookiecutter to prompt the user for all the necessary information to do the proper replacements in the files.  


# Creating a custom template
Since SAM uses cookiecutter to generate a project you can use any of the capabilities described in their [documentation](https://cookiecutter.readthedocs.io/en/2.0.2/)

## 1. Create project structure
You have to give your project the necessary folder structure, as well as include any files needed within these folders. Usually you already have a project in place when creating a template, you can simply copy all of the files and folders from that existing project into your template project.

## 2. Identify repleaceable values
Now you need to identify what values you need to be entered by the person that will be initializing the project. These are things that will vary from project to project. For example, if you are working with a serverless API you could provide values for DynamoDB table name, API stage name, Stack Name, Lambda Function Runtime.  
With the attributes identified you will now replace what you currently have with something that looks like this `{{ cookiecutter.project_name }}`, this is what cookiecutter will be scanning for to be able to replace the values with the users input. This format can be applied to any spot within a file as well as for any folder or file names.

In the screenshot below you can see how I'm using cookiecutter replacement format for a folder name as well as for a string within a file.  
![Screenshot of coockiecutter replacement format examples](/img/how-to-build-a-custom-sam-template/02.jpg)

## 3. Setup cookiecutter metadata
Cookiecutter needs to know which attributes to prompt the user for to be able to use the values as replacements in the template. This is done in a *cookiecutter.json* file at the root of the template project. This is a JSON file that contains all the attributes identified in step #2, the file will look something like this:  
```json
{
  "outputDir": "",
  "service_name": "",
  "dynamodb_table_name": ""
}
```

# Using the custom template
All you have to do now is run the `sam init` command, when asked what the template source is you will choose *Custom Template Location*. You have several options here that depend on where your template is currently located.  
In my example below I used *git*, where I specify the repository url where my template is located:  
![Screenshot of terminal showing custom template example](/img/how-to-build-a-custom-sam-template/03.jpg)

# Summary
We learned how to create a custom template for your serverless applications in 3 simple steps. Having your own template is valuable when your team starts growing and there are new applications getting created very often.  Reducing the amount of time developers have to spend creating folder structures and adding boilerplate functionality to an application is great so the time focused can be spent providing business value.