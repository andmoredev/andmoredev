+++
title = "Step Functions without ASL? Welcome Lambda Durable Functions"
date = 2025-12-07T00:00:00-00:00
draft = true
description = "I've said regularly that AWS Step Functions is my favorite service, this all might be changing with the introduction of Lambda Durable Functions, read along to understand why."
tags = ["AWS", "Serverless", "Lambda"]
[[images]]
  src = "img/layerless-esbuild-lambda/title.png"
  alt = "Image of a person removing a jacket into a stack of other jackets the person has already removed"
  stretch = "stretchH"
+++

Last week during AWS re:Invent, AWS CEO [Matt Garman](https://www.linkedin.com/in/mattgarman/) announced a new feature within AWS Lambda called durable functions. These are a different type of Lambda function that allow you to execute multi-step workflows within a Lambda function handler. What does this mean? You can run similar functionality to what we've typically used AWS Step Functions for. But instead of using Amazon State Language for configuring the workflow, you can use familiar code and dependencies.

It's no secret that AWS Step Functions has been my favorite AWS service for years. Since this new announcement might put my loyalty to the test, I've decided to put them one versus the other to see how they compare in different categories.

## Terms and Concepts
Let's go over a few concepts around durable functions and what their Step Functions equivalent is.

* **Durable Function** - It's a regular Lambda function that can pause and resume by making use of a checkpoint and replay mechanism.
* **Checkpoint** - When the function needs to wait for a callback or sleeps, it will add a checkpoint that persists the current state and stop it's execution.
* **Replay** - Once the workflow is ready to resume execution, it will pick up from the last checkpoint, instead of running through the whole workflow again.
* **Durable Execution** - The complete lifecycle of a durable function. From the moment it is triggered to when it has completed all of the steps defined and is closed.  
* **Durable Context** - The context that is provided to the Lambda handler. This contains the methods for the durable operations.
* **Durable Operations** - These are the operations that allow us to create a workflow within a Lambda function. There are several that we'll go over here.
  * Steps - These run business logic and contain built-in retries and automatic checkpointing. These are defined using `context.step()`
  * Wait States - Planned pauses that will make the function stop running until it is resumed. This can be used for waiting a specific amount of time, wait for an external callback, or for other specific conditions. These are defined using `context.wait()` or `context.waitForCallback()`. There is another special wait state that can be defined by `waitForContition()`, this is essentially polling until a specific condition is met.
  * Parallel - There will be situations where you want to optimize for speed and would like to run things concurrently to reduce the time of execution. The parallel operation is used for that, and is defined by calling `context.parallel()`.
  * Iterations - To process an array of items in a loop you will have to use the `context.map()` operation.
  * Invoke other Lambda functions - You can invoke external Lambda functions from within a workflow. To do this you will call the `context.invoke` function.
  * Nested Operations - To run operations as a child of another operation you will call `context.runInChildContext()`.

## How to crete a a Lambda durable function

This is probably the best thing about all of this. Most of the setup is done the same way as a regular Lambda function, BECAUSE IT'S THE SAME RESOURCE!. In order to enable the durable context for the function we need to do one change in our IaC and another one in the code handler. Let's go over these.

### Infrastructure Setup
To set it up in SAM all you need to do is set the `DurableConfig` object

```yaml
  ExampleFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      CodeUri: workflows/example/
      DurableConfig:
        ExecutionTimeout: 10
        RetentionPeriodInDays: 1
```

Let's go over these two properties in more detail:
* **ExecutionTimeout** - This is the total amount of time in seconds for a full durable execution. This can be up to a full year.
* **RetentionPeriodInDays** - This is the amount of days the durable execution history is stored for after it is closed.

### Code Handler Setup

To make this simpler on us, AWS has provided a new sdk called `@aws/durable-execution-sdk-js`. This contains everything that you'll need to build workflows in a Lambda function. But, right now, the only thing we need for us to have the durable context provided is to wrap our *handler* with the `withDurableExecution` function.
Yes, that is all!

```javascript
import { withDurableExecution } from '@aws/durable-execution-sdk-js';

export const handler = withDurableExecution(async (event, context) => {
  // Your handler code
})
```

With those things in place we can now define the workflow.

### Durable Operations

Now I want to go through the different durable operations we can run and how they look. If you want to look at the full working example, I have it all defined and deployable in [this GitHub location](https://github.com/andmoredev/TrackFlow/tree/main/workflows/durable-function-example).

#### Step

At its most basic level, we have the `step`. This will create a checkpoint after it's run and you can define any logic you want within the step. I recommend splitting the actual business logic to be outside of the workflow definition, to keep it clean and readable. This makes it easy to follow when there are issues with your workflow and you can easily find the step that broke.

```javascript
  // Step 1: Initial step operation - process input data
  const workItems = await context.step('processInputData', async () => {
    return processData(event.inputData);
  });
```

#### Human in the loop

Whenever you want to wait for a human action you will use the `waitForCallback` operation.

```javascript
  // Step 2: Wait for callback operation - pause for external event
  // Wait for external callback with timeout handling
  const callbackResult = await context.waitForCallback(
    "wait-for-external-callback",
    async (callbackId, ctx) => {
      // Submit callback ID to external system (simulated)
      ctx.logger.info(`Callback ID ${callbackId} submitted to external system`);
      // In real implementation, this would call an external API
      // await submitToExternalAPI(callbackId);
    },
    { timeout: { minutes: 60 } } // 1 hour timeout
  );
```

This will give us a `callbackId` that we can use to resume our workflow. When testing this can easily be done from within the AWS Console, but in real life you will need to do this programatically. Thankfully the SDK provides the `SendDurableExecutionCallbackSuccessCommand` and `SendDurableExecutionCallbackFailureCommand` in the Lambda client, that can be used to resume the workflow. You can also specify a *timeout* so the workflow is not waiting forever.

Once the command is sent the workflow will resume and continue processing.

#### Wait

If you ever need to simply wait a specific amount of time, you have the `wait` operation where you specify the amount of time you want to wait.

```javascript
  // Step 3: Simple wait operation - demonstrate time-based wait
  await context.wait({ seconds: 5 }); // Wait for 5 seconds
```

#### Parallel Processing

When you have several operations that can be handled independently, you would want to run them in parallel to reduce the total processing time. That's where the `parallel` operation comes in.

```javascript
  // Step 4: Parallel operations - process multiple work streams concurrently
  const parallelResults = await context.parallel([
    async (ctx) => ctx.step('parallelTask1', async () => {
      return await performDataValidation(workItemsCount);
    }),
    async (ctx) => ctx.step('parallelTask2', async () => {
      return await performDataEnrichment(workItemsCount);
    }),
    async (ctx) => ctx.step('parallelTask3', async () => {
      return await performQualityCheck();
    })
  ]);
```

This will attempt to run all three operations in parallel. If you are worried about putting too much load to another service, you can set `concurrency` limits so not everything processes at once.

#### Iterate Arrays

I always say that programming is all about `if`s and `for`s. So how do you handle the `for` situation when working with durable functions? We have the `map` operator!

```javascript
  // Step 5: Map operation - iterate over collection with checkpoints
  const mapResults = await context.map(workItems, async (ctx, item, index) => {
    return await ctx.step(`processItem-${index}`, async () => {
      const { processedItem, processingTime } = processWorkItem(item, index);

      // Simulate processing time based on priority
      await new Promise(resolve => setTimeout(resolve, processingTime));

      return processedItem;
    });
  });
```

You might be asking yourself, why not just use a regular `for` loop? The answer is, CHECKPOINTS. By using the `map` operation you get the benefit of having Lambda manage the checkpoint and the current state of where things left off. This helps immensly if it didn't process the array completely.

#### Wait for condition
There is also a special wait operation that waits for a specific condition to be met. For this you will use the `waitForCondition` operation. You can think of this operation as a *do/while*, where it will keep iterating until the *while* condition is met.

```javascript
// Step 6: Wait for condition - poll until external system is ready
  const conditionResult = await context.waitForCondition(
    async (state, ctx) => {
      const readinessCheck = await checkSystemReadiness();
      return {
        ...state,
        ready: readinessCheck.ready
      };
    },
    {
      initialState: {
        ready: false,
      },
      waitStrategy: (state) =>
        state.ready
          ? { shouldContinue: false }
          : { shouldContinue: true, delay: { seconds: 3 } }
    }
  );
```

This has two parameters:
1. WaitForConditionCheckFunc - Function that will check the current state and return the updated state.
1. WaitForConditionConfig - Holds the configuration for the initial state and the wait strategy that is used to determine when it should continue and how long it should delay before it triggers the WaitForConditionCheckFunc again.

#### Invoke Lambda Function

As much as you try to avoid it, you will have a reason to invoke a separate Lambda function that does it's own processing. This will be done by using the `invoke` operation. All you need to do is give it the function's ARN and the expected payload. 

```javascript
  // Step 7: Invoke another Lambda function
  const invokePayload = createInvokePayload(workItems.length, context.executionId);
  const invokeResult = await context.invoke(
    'invoke-hello-world',
    process.env.HELLO_WORLD_FUNCTION_ARN,
    invokePayload
  );
```

#### Child contexts

The `runInChildContext` operation is used to create an isolated execution context for a group of operations. These have their own checkpoint logging and can have multiple steps, waits and other operations. This is treated as a single unit for retry and recovery. Use these when you want to organize complex workflows, implement sub-workflows or isolate operations that should retry together.

```javascript
  // Step 8: Run operations in child context for isolation
  const childContextResult = await context.runInChildContext('isolated-operations', async (childCtx) => {
    // These operations run in isolation with their own checkpoint log
    const metadata = await childCtx.step('processMetadata', async () => {
      return await processMetadataInChild(context.executionId, workItems.length);
    });

    const validation = await childCtx.step('validateConfiguration', async () => {
      return await validateConfigurationInChild();
    });

    return {
      metadata,
      validation,
      childExecutionId: childCtx.executionId,
      completedAt: Date.now()
    };
  });
```

### Invoking & Troubleshooting

Once your durable function is deployed you will want to invoke the function. Well, surprise, surprise, this is done exactly the same as how you invoke any Lambda function, which means you can use all of your regular event sources like API Gateway, EventBridge, SQS. All of what I'm about to do can be done using the CLI, but for demo reasons I will be using the console so we can get better visualizations.

You will now see this new *Durable executions* tab in the Lambda console. 

![Durable executions tab](/img/lambda-durable-functions/durable-executions-tab.png)

To get a new execution in the list, I simply invoked the function using the normal testing tool in the Lambda console.

Now let's look at the details of the execution

![Execution details](/img/lambda-durable-functions/execution-details.png)

This view should look very similar to what we've seen in AWS Step Functions, it doesn't have the nice workflow diagram those. There are two sections here:

1. Durable operations - here we will see all of the operations that have executed. In our example above it only shows two items because it has the human-in-the-loop step and waiting for someone to respond with the callback Id. We will do that in a second.
1. Event history - a more verbose list of what has happened. This is where you will be able to track any errors that happened in a specific step and just get a broader view of everything that executed.

Now let me send the success message for the callback. I am going to be doing this directly from the console as shown below. If you are looking for a way to do this programatically, I've included a [script here](https://github.com/andmoredev/TrackFlow/blob/main/scripts/resume-workflow.mjs). This script will take in the callback Id and send a success message to continue the flow.

![Callback send success button](/img/lambda-durable-functions/callback-success.png)

This will now continue and execute all of the steps we have defined in our workflow.

![Full execution](/img/lambda-durable-functions/full-execution.png)

You'll notice that some operations have an explicit name and others have a random ID. The reason for this is that I did not explicitly specify the name for all the operations, so it will generate a random one for us.

Let's walk through the operations.

* WaitForCallback - We can see that the duration was 9 minutes and 43 seconds. That is how long I took to write a few of the paragraphs above to then press the "Send success" button.
* Wait - As expected this took the 5 seconds we had configured it for.
* Parallel - This operation groups all of the sub operations under it, making it easier to track.

![Parallel operation](/img/lambda-durable-functions/parallel.png)

* Map - Similarly to the parallel step, it will group all of the iterations under it.

![Map operation](/img/lambda-durable-functions/map.png)

* WaitForCondition - The special emphasis on this one is that it has a 2 on the Retries. This means that it had to iterate two times before the condition was met.
* ChainedInvoke - Not too much to see for this step. It simply invoked the Lambda function and continued on.
* RunInChildContext - This one also groups all the child operations under it. 
![Run in child context operation](/img/lambda-durable-functions/runinchildcontext.png)

This allows you to see the different operations and how they look in the console. You can get this information by using the CLI or the SDK as well and create your own visualizations if necessary.

## Wrap Up
