+++
title = "Step Functions without ASL? Welcome Lambda Durable Functions"
date = 2025-12-17T00:00:00-00:00
draft = false
description = "I've regularly said that AWS Step Functions is my favorite service, this all might be changing with the introduction of Lambda Durable Functions. We'll be going on a deep dive into durable functions and how they work."
tags = ["AWS", "Serverless", "Lambda"]
[[images]]
  src = "img/lambda-durable-functions/title.png"
  alt = "Distracted boyfriend meme. The boyfriend is looking at a Lambda durable function while the angry girlfriend has the Step Functions logo as the face."
  stretch = "stretchH"
+++

During re:Invent, AWS announced a new feature within AWS Lambda called durable functions. These are the same Lambda functions we all love, but they let you run multi-step workflows by keeping checkpoints and state. What does this mean? You can run similar functionality to what we’ve typically used AWS Step Functions for. But instead of using Amazon State Language, you can use familiar code and dependencies.

## Terms and Concepts

Let’s go over a few concepts around durable functions.

* **Durable Function** - It’s a regular Lambda function that can pause and resume by making use of a checkpoint and replay mechanism.
* **Checkpoint** - When the function executes a step or needs to wait for a callback, it will add a checkpoint that persists the current state and stops its execution.
* **Replay** - Once the workflow is ready to resume execution, it will pick up from the last checkpoint rather than running the whole workflow again.
* **Durable Execution** - The complete lifecycle of a durable function. From the moment it is triggered until it completes all defined steps and is closed.
* **Durable Context** - The context that is provided to the Lambda handler. This contains the methods for the durable operations.
* **Durable Operations** - These are the operations that allow us to create a workflow within a Lambda function. Each step has built-in retries and automatically creates checkpoints when it runs. Let’s go over each of the operations at a high level.
    * **Steps** - Run business logic and are defined by using the `context.step()` operator.
    * **Wait States** - Planned pauses that cause the function to stop running until they are resumed. This can be used to wait for a specific amount of time, for an external callback, or for other specific conditions. These are defined using `context.wait()` , `context.waitForCallback()` or `context.waitForContition()`.
    * **Parallel** - There will be situations where you want to optimize for speed and run things concurrently to reduce execution time. The parallel operation is used for that, and is defined by using the `context.parallel()` operator.
    * **Iterations** - To process an array of items in a loop, you will have to use the `context.map()` operation.
    * **Invoke other Lambda functions** - You can invoke external Lambda functions from within a workflow. To do this, you will call the `context.invoke()` operator.
    * **Nested Operations** - To run operations as a child of another operation, you will call `context.runInChildContext()`.

## How to create a Lambda durable function

This is probably the best thing about all of this. Most of the setup is done the same way as a regular Lambda function, BECAUSE IT’S THE SAME RESOURCE!. To enable a durable context for the function, we need to make one change to our IaC and another to the code handler. Let’s go over these.

### Infrastructure Setup

To set it up in SAM, all you need to do is set the DurableConfig object.

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

Let’s go over these two properties in more detail:

* **ExecutionTimeout** - This is the total time, in seconds, for a full durable execution. This can be up to a full year.
* **RetentionPeriodInDays** - The number of days the durable execution history is retained after it is closed.

### Code Handler Setup

To make this simpler for us, AWS has provided a new SDK called @aws/durable-execution-sdk-js. This contains everything that you’ll need to build workflows in a Lambda function. But right now, the only thing we need to have a durable context is to wrap our handler with the `withDurableExecution` function.
Yes, that is all!

```javascript
import { withDurableExecution } from '@aws/durable-execution-sdk-js';

export const handler = withDurableExecution(async (event, context) => {
  // Your handler code
})
```

With those things in place, we can now define the workflow.

### Durable Operations

Now I want to go through the different durable operations in detail and show them in a working example. If you want to follow along, I have everything defined and deployable in [this GitHub location](https://github.com/andmoredev/durable-functions/blob/main/workflows/durable-function-example/index.mjs).

#### Step

At its most basic level, we have the step. This will create a checkpoint after it’s run, and you can define any logic you want within the step. I recommend splitting the actual business logic outside of the workflow definition to keep it clean and readable. This makes it easy to follow when there are issues with your workflow, and you can find the step that broke.

```javascript
  // Step 1: Initial step operation - process input data
  const workItems = await context.step('processInputData', async () => {
    return processData(event.inputData);
  });
```

#### Human in the loop

Whenever you want to wait for a human action, you will use the `waitForCallback` operation.

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

This will give us a `callbackId` that we can use to resume our workflow. When testing, you can resume the workflow from the AWS Console, but in real life, you will need to do this programmatically. Thankfully, the SDK provides the SendDurableExecutionCallbackSuccessCommand and SendDurableExecutionCallbackFailureCommand in the Lambda client, which can be used to resume the workflow. You can also specify a timeout so the workflow does not wait forever.

Once the command is sent the workflow will resume and continue processing.

#### Wait

If you ever need to wait a specific amount of time, you can use the `wait` operation, which lets you specify the amount of time you want to wait.

```javascript
  // Step 3: Simple wait operation - demonstrate time-based wait
  await context.wait({ seconds: 5 }); // Wait for 5 seconds
```

#### Parallel Processing

When you have several operations that can be handled independently, you might want to run them in parallel to reduce the total processing time. That’s where the `parallel` operation comes in.

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

This will attempt to run all three operations in parallel. If you are worried about putting too much load on another service, you can set `concurrency` limits so that not everything is processed at once.

#### Iterate Arrays

I always say that programming is all about `ifs` and `fors`. So how do you handle the `for` situation when working with durable functions? We have the `map` operator!

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

You might be asking yourself, Why not just use a regular `for` loop? The answer is CHECKPOINTS. By using the `map` operation, you get the benefit of having Lambda manage the checkpoint and the current state of where things left off. This helps immensely if it didn’t process the array completely.

#### Wait for condition

There is also a special wait operation that waits for a specific condition to be met. For this, you will use the `waitForCondition` operation. You can think of this operation as a `do/while`, where it will keep iterating until the `while` condition is met.

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

1. **WaitForConditionCheckFunc** - Function that will check the current state and return the updated state.
2. **WaitForConditionConfig** - Holds the configuration for the initial state and the wait strategy used to determine when it should continue and how long to delay before triggering WaitForConditionCheckFunc again.

#### Invoke Lambda Function

As much as you try to avoid it, you will have a reason to invoke a separate Lambda function that does its own processing. This will be done by using the `invoke` operation. All you need to do is give it the function’s ARN and the expected payload.

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

The `runInChildContext` operation creates an isolated execution context for a group of operations. These have their own checkpoint logging and can have multiple steps, waits and other operations. This is treated as a single unit for retry and recovery. Use these when you want to organize complex workflows, implement sub-workflows or isolate operations that should retry together.

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

Once your durable function is deployed, you will want to invoke the function. Well, surprise, surprise, this is done exactly the same way you invoke any Lambda function, which means you can use all your regular event sources like API Gateway, EventBridge, and SQS. All of what I’m about to do can be done using the CLI, but for demo reasons, I will be using the console so we can get better visualizations.

When the Lambda is deployed with the `DurableConfig` you will see a new Durable executions tab in the Lambda console.

![Durable executions tab](/img/lambda-durable-functions/durable-executions-tab.png)

To create a new execution, I invoked the function using the standard testing tool in the Lambda console.

Now, let’s look at the details of the execution.

![Execution details](/img/lambda-durable-functions/execution-details.png)

This view should look very similar to what we’ve seen in AWS Step Functions. It doesn’t have the nice workflow diagram, though. There are two sections here:

1. **Durable operations** - here we will see all the operations that have been executed. In our example above, only two items appear because it has the human-in-the-loop step and is waiting for someone to respond with the callback ID. We will do that in a second.
2. **Event history** - a more verbose list of what has happened. This is where you can track errors that occurred in a specific step and get a broader view of everything that was executed.

Now, let me send the success message for the callback. I will be doing this directly from the console, as shown below. If you are looking for a way to do this programmatically, I’ve included a [script here](https://github.com/andmoredev/durable-functions/blob/main/scripts/resume-workflow.mjs). This script will take in the callback ID and send a success message to continue the flow.

![Callback send success button](/img/lambda-durable-functions/callback-success.png)

This will now continue and execute all of the steps we have defined in our workflow.

![Full execution](/img/lambda-durable-functions/full-execution.png)

You’ll notice that some operations have explicit names and others have random IDs. The reason is that I did not explicitly specify a name for all the operations, so it will generate a random one for us.

Let’s walk through the operations.

* **WaitForCallback** - The duration was 9 minutes and 43 seconds. That is how long it took to write a few of the paragraphs above, and then press the “Send success” button.
* **Wait** - As expected, this took the 5 seconds we had configured it for.
* **Parallel** - This operation groups all sub-operations under it, making them easier to track.

![Parallel operation](/img/lambda-durable-functions/parallel.png)

* **Map** - Similarly to the parallel step, it will group all of the iterations under it.

![Map operation](/img/lambda-durable-functions/map.png)

* **WaitForCondition** - The special emphasis here is that it has a 2 in Retries. This means it had to iterate twice before the condition was met.
* **ChainedInvoke** - Not too much to see for this step. It simply invoked the Lambda function and continued on.
* **RunInChildContext** - This one also groups all the child operations under it.
![Run in child context operation](/img/lambda-durable-functions/runinchildcontext.png)

This allows you to see the different operations and how they look in the console. You can get this information using the CLI or the SDK, and create your own visualizations if necessary.

## Wrap Up

Woah! That was a lot! We went through the setup to enable and deploy a durable function, as well as how to build a workflow that uses all of the currently supported durable operations.

The fact that I am now able to create long-running workflows where I can include any npm dependency directly without having to explicitly call external Lambda functions is a huge win compared to what we've been doing with Step Functions.

I am going to keep playing around with this and will be doing a side-by-side comparison with AWS Step Functions. Stay tuned!

Until next time!

Andres Moreno