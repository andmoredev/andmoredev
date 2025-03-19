+++
title = "Log buffering with Lambda Powertools"
date = 2025-03-18T00:00:00-00:00
draft = false
description = ""
tags = ["AWS", "Serverless", "Lambda", "Powertools"]
[[images]]
  src = "img/update-commonjs-to-esm/title.jpg"
  alt = ""
  stretch = "stretchH"
+++

The Lambda Powertools team released a new feature that allows your Lambda functions to buffer logs. That sounded cool, but I wasn’t really understanding how it would work and how logs would be presented in CloudWatch. That is why I decided to try it out and provide a visual example of how it looks with different configuration options.

We’ll be going through the following:
* What is log buffering?
* Why buffer logs?
* Configuration options
* Things to keep in mind
* Demo
* Wrap up

### What is log buffering?
This is what the Lambda Powertools documentation says:

Log buffering enables you to buffer logs for a specific request or invocation. Enable log buffering by passing **logBufferOptions **when initializing a Logger instance. You can buffer logs at the **WARNING**, **INFO**,  **DEBUG**, or **TRACE** level, and flush them automatically on error or manually as needed.

The three main terms from that explanation are *buffering, level, *and *flush. *Let’s explain each of these in the context of log buffering.

* **Buffering** - this means that the Lambda function will hold the logs in memory without sending them to CloudWatch unless it is necessary.
* **Level** - The log level at which you want to start buffering the logs. Below are all the supported log levels and their numeric value.

|  **Level**<br/> | **Numeric Value**<br/> |
|-----|-----|
|  TRACE<br/> | 6<br/> |
|  DEBUG<br/> | 8<br/> |
|  INFO<br/> | 12<br/> |
|  WARN<br/> | 16<br/> |
|  ERROR<br/> | 20<br/> |
|  CRITICAL<br/> | 24<br/> |
|  SILENT<br/> | 28<br/> |

You can choose which level to start buffering at. For example, if you configure the log buffer to do it at the INFO level, it will buffer those logs and any log levels with a lower value. In this case, it will also buffer TRACE and DEBUG.
* **Flush** - This means the logs that are buffered will get sent to CloudWatch. There are different ways you can tell the Lambda to send the logs to be able to successfully troubleshoot in case of errors. We’ll be covering these options in the next section.

### Why buffer logs?
At this point you might be asking yourself, why do I want to buffer logs. And the reason not be immediately apparent. Most of the time logs are only needed when your application hit an erro. This means that we are constantly logging things for all the requests and processing that are successful and will be producing logs that will not be seen by anyone. By enabling logging buffer you can make sure to only print the logs whenever a specific scenario or error was hit, allowing you to reduce the amount of logs produced therefore reducing the noise but even better reducing your logging costs.
**### 
**
**### Configuration options**
There are a few pieces of configuration for buffer logging that can be set when you initialize a logger to be able to handle things according to what you need.

* enabled - this should be pretty straightforward. Log buffering is disabled by default; in order to start using it, this parameter will need to be set to *true.*
* maxBytes - this is the max amount of bytes the buffer will hold in memory. If you happen to hit the max limit, the buffer will start dropping older logs to make room for the new ones. This is a protection measure so your Lambda function doesn't run out of memory. The logger will be nice enough to let you know it did this whenever the logs are flushed. This defaults to 20480 bytes.
* bufferAtVerbosity - this is where you will configure at which log level you want to start buffering, as I had mentioned before. Whatever value you configure will be buffered and any level with a lower numeric value. The default log level set is DEBUG.
* flushOnErrorLog - This is a boolean value that tells the buffer to log whenever it encounters a *logger.error *call. When writing Lambda functions, we typically have a try/catch statement where we log any errors that get thrown. This helps automatically get the logs flushed when an error is caught and logged.

But now the question is, what happens if I didn't catch and log an error? Will I not be able to see the buffered logs? Thankfully, the Lambda Powertools team thought about this and they allow you to flush the logs in two other ways:
*  **flushBufferOnUncaughtError*** - *This option is only allowed when you are injecting the Lambda context into the logger. You can use this to flush logs when there is an uncaught error. What this property will do is to flush the logs whenever the Lambda function encounters any error and is not caught.
* **flush_buffer function** - You might want to flush the logs whenever a specific code path is hit or other scenarios that are specific for your use case. To do this, the logger has a new function called *flush_buffer *that will take care of sending all of the buffered logs into CloudWatch manually.

**### Demo**
Let's see this in action!

**Enable log buffering**
To enable this feature all we have to do is set the enabled property to true in the logBufferOptions object for the Logger constructor as shown below:
[code block showing instantiation]

Now to test we will add all the different types of logs to our Lambda handler like this:
[Code block showing Lambda handler]

If we run the Lambda function and view the logs produced in CloudWatch, we get this:
[Image showing the cloudwatch logs it should only show logs for debug and below]

As you can see from the image, the DEBUG and TRACE logs are not shown since they are buffered at that log level by default.

**Set log level**
Let’s now configure the logger to buffer at the WARN level.
[Code block showing config]

If we run our Lambda function we now get less logs since we are buffering at a higher level.
[Image showing less logs in CloudWatch]

**Flushing logs**
We’ll first configure our logger to flush logs on uncaught errors and manually throwing an exception in our code.
[Code blocks showing the changes]

Looking at the logs emitted we see how they all made it and by the order they are in you can see when it flushed the buffer because all of the ones with the log level are at the end (ANDRES confirm this as they may still be in correct order as they persist the timestamp)
**
**
Now we are going to configure it to flush the buffered logs on error log. Lets remove our exception being thrown and change the logger options.
[Code block showing configuration]

Viewing our logs we can see that all of them show the same as the previous test.
[image if CloudWatch showing errors]

Last but not least, lets flush the logs manually. To do this lets first disable the option to flush on error log and add a call to this function.

Taking a look at the logs we can see only the ones buffered before the flush make it.

**Hitting max buffer size**
Now let’s make our buffer hit the max size limit by adding a loop that is logging a lot.
[code block]
**
**
Let’s also add an extraordinarily large log directly into our code.
[code block]

Looking in CloudWatch for these we can see the errors that are letting us know that old logs were dropped and that a large log was not buffered because it exceeded the size.
[images showing the messages]

### Wrap up
We talked about what log buffering is, why you would use it, how to use it and saw it in action (by screenshots 😊). I am always impressed with the great functionality the Powertools team adds to this project. It is by far my favorite package out there and greatly simplifies following best practices for Lambda function development. 
There are many other features and I recommend looking at their documentation so you can identify which features you can apply to your projects.

Let me know what you think about this feature!

Until next time!

Andres Moreno
