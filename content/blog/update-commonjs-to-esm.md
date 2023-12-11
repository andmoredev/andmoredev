+++
title = "How to update Lambda Functions from Common JS to ECMAScript"
date = 2023-12-11T00:00:00-00:00
draft = false
description = "Follow these 3 steps to update your Common JS Lambda Functions to ECMAScript Modules (ESM)"
tags = ["AWS", "Serverless"]
[[images]]
  src = "img/update-commonjs-to-esm/title.jpg"
  alt = "Cartoon image of a laptop with gears and a person with a big wrench"
  stretch = "stretchH"
+++

Version 5.0.0 of the [@middy/core](https://www.npmjs.com/package/@middy/core) package no longer supports Common JS in favor of ECMAScript Modules (ESM). ESM improves performance by 2X as mentioned [in this post](https://middy.js.org/docs/upgrade/4-5/). Seeing this change prompted me to investigate what needs to be updated to get Lambda functions to be ESM to not fall behind on my dependency versions. In this post, we'll go through the 3 steps needed to go from Common JS to ESM.

# 1. Designate Lambda functions as ECMA Script Modules
There are two ways you can designate a Lambda function as an ECMA Script Module.

### Update package.json
You can add the **type** attribute with a value of **module** to the package.json as shown below.  

![package.json diff showing type attribute as module](/img/update-commonjs-to-esm/01.jpg)

### Update file extension
You can also set the extension of the files to be *.mjs* instead of *.js* or *.cjs*

# 2. Update how you include dependencies
In Common JS dependencies are included using the *require* keyword, in ECMAScript with the *import* keyword. Below we can see the code change needed to include dependencies in ESM.  

![Diff file showing the difference between require and import for including modules](/img/update-commonjs-to-esm/02.jpg)


# 3. Update how the *handler* and other functions are exported
The way you export functions in ESM is different than how it is done for Common JS. The export will have to change from `exports.functionName` to `export const functionName` as shown below.  

![Diff file showing how exporting functions changes](/img/update-commonjs-to-esm/03.jpg)

# Last words
With these three changes you are now done, these are simple steps to take but might get more complicated depending on your project structure. If you are applying this to a more complicated project I recommend having test automation (unit tests, contract tests, etc) to verify no bugs are getting introduced.
