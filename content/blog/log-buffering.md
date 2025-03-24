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

> Log buffering enables you to buffer logs for a specific request or invocation. Enable log buffering by passing **logBufferOptions** when initializing a Logger instance. You can buffer logs at the **WARNING**, **INFO**,  **DEBUG**, or **TRACE** level, and flush them automatically on error or manually as needed.

The three main terms from that explanation are *buffering, level, *and *flush. *Let’s explain each of these in the context of log buffering.

* **Buffering** - this means that the Lambda function will hold the logs in memory without sending them to CloudWatch unless it is necessary.
* **Level** - The log level at which you want to start buffering the logs. You can choose which level to start buffering at. For example, if you configure the log buffer to do it at the INFO level, it will buffer those logs and any log levels with a lower value. In this case, it will also buffer TRACE and DEBUG. Below are all the supported log levels and their numeric value.

|  **Level**<br/> | **Numeric Value**<br/> |
|-----|-----|
|  TRACE<br/> | 6<br/> |
|  DEBUG<br/> | 8<br/> |
|  INFO<br/> | 12<br/> |
|  WARN<br/> | 16<br/> |
|  ERROR<br/> | 20<br/> |
|  CRITICAL<br/> | 24<br/> |
|  SILENT<br/> | 28<br/> |  

* **Flush** - This means the logs that are buffered will get sent to CloudWatch. There are different ways you can tell the Lambda to send the logs to be able to successfully troubleshoot in case of errors. We’ll be covering these options in the next section.

### Why buffer logs?
At this point you might be asking yourself, why do I want to buffer logs. And the reason not be immediately apparent. Most of the time logs are only needed when your application hit an erro. This means that we are constantly logging things for all the requests and processing that are successful and will be producing logs that will not be seen by anyone. By enabling logging buffer you can make sure to only print the logs whenever a specific scenario or error was hit, allowing you to reduce the amount of logs produced therefore reducing the noise but even better reducing your logging costs.

### Configuration options
There are a few pieces of configuration for buffer logging that can be set when you initialize a logger to be able to handle things according to what you need.

* enabled - this should be pretty straightforward. Log buffering is disabled by default; in order to start using it, this parameter will need to be set to *true.*
* maxBytes - this is the max amount of bytes the buffer will hold in memory. If you happen to hit the max limit, the buffer will start dropping older logs to make room for the new ones. This is a protection measure so your Lambda function doesn't run out of memory. The logger will be nice enough to let you know it did this whenever the logs are flushed. This defaults to 20480 bytes.
* bufferAtVerbosity - this is where you will configure at which log level you want to start buffering, as I had mentioned before. Whatever value you configure will be buffered and any level with a lower numeric value. The default log level set is DEBUG.
* flushOnErrorLog - This is a boolean value that tells the buffer to log whenever it encounters a *logger.error *call. When writing Lambda functions, we typically have a try/catch statement where we log any errors that get thrown. This helps automatically get the logs flushed when an error is caught and logged.

But now the question is, what happens if I didn't catch and log an error? Will I not be able to see the buffered logs? Thankfully, the Lambda Powertools team thought about this and they allow you to flush the logs in two other ways:
*  **flushBufferOnUncaughtError*** - *This option is only allowed when you are injecting the Lambda context into the logger. You can use this to flush logs when there is an uncaught error. What this property will do is to flush the logs whenever the Lambda function encounters any error and is not caught.
* **flush_buffer function** - You might want to flush the logs whenever a specific code path is hit or other scenarios that are specific for your use case. To do this, the logger has a new function called *flush_buffer *that will take care of sending all of the buffered logs into CloudWatch manually.

### Demo
Let's see this in action!

**Enable log buffering**
To enable this feature all we have to do is set the enabled property to true in the logBufferOptions object for the Logger constructor as shown below:
```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true
  }
});
```

Now to test we will add all the different types of logs to our Lambda handler like this:
```javascript
    logger.trace('Trace Log');
    logger.debug('Debug Log');
    logger.info('Info Log');
    logger.warn('Warn Log');
    logger.error('Error Log');
    logger.critical('Critical Log');

```

If we run the Lambda function and view the logs produced in CloudWatch, we get this:
![Cloudwatch Logs](/img/log-buffering/logs-1.png)

The results were a bit unexpected to me, but then I went back and looked at the default values for the logger options to try and understand this better. There are two default values to keep in mind for this example, the **flushOnErrorLog** is enabled by default and the default log level for the buffer is DEBUG. With this let's understand why the logs show in this specific order.

* The TRACE and DEBUG logs are added to the buffer because of the log level setting.
* The INFO and WARN logs are printed as they are added since they are not included in the buffer because of the log level setting.
* The TRACE and DEBUG logs are now shown because we hit the ERROR log. Meaning logs are flushed before the ERROR log is printed.
* Finally the ERROR and CRITICAL logs are printed.

The order of the logs are different than the order of them in the code. The reason for this is because by default the logs are buffered at the DEBUG level which means the TRACE and DEBUG logs were buffered, so we first see the INFO and WARN. After that the logs are automatically flushed by the ERROR log which is why we see the TRACE and DEBUG logs that were buffered and we see the last two logs

If we were to run the same test but in this scenario we don't log an error we can really appreciate the value of the log buffer.

![CloudWatch Logs](/img/log-buffering/logs-2.png)

From the image above we can see that only the non-buffered logs made it to CloudWatch, meaning that if there is no error we wont see or get charged for the buffered logs.

**Set log level**
Let’s now configure the logger to buffer at the WARN level.
```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true,
    bufferAtVerbosity: 'WARN'
  }
});
```

If we run our Lambda function we now get less logs since we are buffering at a higher level.
![CloudWatch Logs](/img/log-buffering/logs-3.png)

**Hitting max buffer size**
Now let’s make our buffer hit the max size limit by intentionally overloading the buffer. For this example we are going to manually flush the logs by calling the function, this means we'll need to turn off the flushOnErrorLog property and to make this test easier, I will set the maxBytes property to 1000.

```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true,
    bufferAtVerbosity: 'WARN',
    flushOnErrorLog: false,
    maxBytes: 1000
  }
});
```

And for the logging code, we'll do a simple for loop and manually flush at the end.
```javascript
for (let index = 0; index < 500; index++) {
  logger.trace('Trace Log');
  logger.debug('Debug Log');
  logger.info('Info Log');
  logger.warn('Warn Log');
  logger.error('Error Log');
}

logger.flushBuffer();
```

When we run this Lambda Powertools will let us know that we've exceeded the size of the buffer and that some of the logs were lost to make room for the newer ones.

![CloudWatch Logs](/img/log-buffering/logs-3.png)

And what happens if a single log exceeds the size of the buffer? Well let's try it. To do this I'm going to set the maxBytes property to just 1 byte.
```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true,
    bufferAtVerbosity: 'WARN',
    flushOnErrorLog: false,
    maxBytes: 1
  }
});
```

If we run the same code as above, we will see a message telling us the log was not able to be buffered because it was too big.

![CloudWatch Logs](/img/log-buffering/logs-3.png)

### Wrap up
I'm very excited to start using this in projects and see how the CloudWatch pricing can be impacted. The best part of this is that Lambda Powertools is taking care of most of the heavy lifting and all we have to do is configure a few properties to have it behave the way we would want to. I really want to thank the Lambda Powertools team as they keep impressing me with all the great functionality they keep delivering that only makes our lives easier.

Let me know what you think about this feature!

Until next time!

Andres Moreno
