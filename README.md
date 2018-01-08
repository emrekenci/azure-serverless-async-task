# A serverless approach to async task management
## Overview
Here we present an easy way to:
* Create an API that accepts requests for task completion.
* A queue mechanism to decouple the app that requests the task and the app that completes the task.
* Create a way to manage tasks which could not be completed.

...using serverless technologies on Azure.

This approach can be applied to any scenario where a requested task is completed asynchronously such as:

* Image processing
* File conversion
* Invoice generation
* Sending a non-transactional sms/email

...or any other task which doesn’t have to be completed at the time it was requested.

Here is an overview of what we’ll be covering:
![A serverless approach to async task management](https://github.com/emrekenci/azure-serverless-async-task/raw/master/images/A%20serverless%20approach%20to%20async%20task%20management.png "A serverless approach to async task management")

The exciting thing about this approach isn't the architecture itself but how little **_code & effort_**  it takes to implement this on Azure. The rest of the article describes the implementation details.

## Queueing and dequeueing functions

We'll create two Azure Functions using Visual Studio. We **_could_** code the functions on the portal in Javascript or C# Script, but the samples I'll share are written in C# on Visual Studio.

You can see the full solution for the functions [here](
https://github.com/emrekenci/azure-functions-servicebus-sample).

For some simplified code samples for Service Bus, you can check out [this repository](https://github.com/emrekenci/azure-servicebus-sample).

### HTTP Triggered Function (Queue tasks)

Simply writes the request body to Service Bus Queue as a message. Here is the code:
```csharp
[FunctionName("enqueue")]
public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Function, "post")]HttpRequestMessage req, TraceWriter log)
{
    string requestBody = await req.Content.ReadAsStringAsync();

    string serviceBusConnectionString = Environment.GetEnvironmentVariable("ServiceBusConnectionString", EnvironmentVariableTarget.Process);
    string serviceBusQueueName = Environment.GetEnvironmentVariable("ServiceBusQueueName", EnvironmentVariableTarget.Process);

    IQueueClient queueClient = new QueueClient(serviceBusConnectionString, serviceBusQueueName);

    byte[] messageBytes = Encoding.ASCII.GetBytes(requestBody);

    var message = new Message(messageBytes);

    await queueClient.SendAsync(message);

    return req.CreateResponse(HttpStatusCode.OK);
}
```
The code above assumes your ServiceBus endpoint is already created. You can learn how to setup your ServiceBus Queue [here](
https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues).

Note that as soon as we deploy this function to Azure, we will have a functioning HTTP endpoint that's ready to be used.

Here is what this function **_saves_** us from doing:

* Creating an API project
* Setting up routing and the API endpoints
* Writing API authentication code
* Provisioning a web server or a web app.
 
 ### Queue Triggered Function (Complete tasks)

This function is the worker which receives and completes the tasks in the queue. Whenever a new message is written to the queue, this function is automatically called and passed the message body as a parameter.

We don't need to complete or abandon the message explicitly, the ServiceBusTrigger attribute takes care of that for us. If the function completes without throwing an exception, the message will be deleted from the queue, if the function does throw an exception the message will be abandoned.
```csharp
[FunctionName("dequeue")]
public static void Run([ServiceBusTrigger("myqueue", AccessRights.Manage, Connection = "ServiceBusConnectionString")]string queueMessageContent, TraceWriter log)
{
    try
    {
        log.Info("Dequeued task: " + queueMessageContent);
    }
    catch (Exception e)
    {
        /*
         * The message will be automatically abandoned when the function throws an exception.
         * After the max retry count is exceeded, the message will be moved to the deadqueue.
         * The trigger handles this behaviour. No need to write plumming code.
         */
        log.Error(e.Message, e);

        throw;
    }
}
```

You can see that the function above saves us a good amount of work. No need for:
* Instantiating a ServiceBus Queue client
* Subscribing to the queue messages
* Explicitly completing or abandoning messages.
* Setting up a worker app in a VM/container or deploying & configuring a webjob

Once again, the full code samples are [here](
https://github.com/emrekenci/azure-functions-servicebus-sample).

## Setting up a Logic App to handle errors. (Dead-letter queue listener)

Now we'll create a logic app to deal with problematic tasks. If you're not familiar with Logic Apps, I suggest you [take a look](https://azure.microsoft.com/en-us/services/logic-apps/). It's like IFTTT for Devops on Azure.

If the message/task cannot be completed after a certain number of tries, it will be sent to [the dead-letter queue](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues). In this situation, we would typically code another app that listens to the DLQ, and either fixes the problem with the message or just notifies the team responsible.

As it turns out, there is a way to do this without writing **_a single line of code_** on Azure. Here is how:

![Logic app service bus DLQ trigger](https://github.com/emrekenci/azure-serverless-async-task/raw/master/images/Logic%20App.png "Logic app service bus DLQ trigger")

1. Go to Logic Apps blade on Azure and create a blank Logic App.
2. Select ServiceBus in the list of connectors
You will see two options:
* Service Bus - When a message is received in a queue (auto-complete)
* Service Bus - When a message is received in a queue (peek-lock)

The first option completes(deletes) the dead-letter queue after processing the message. The second one leaves the message in DLQ unless you specify otherwise.

3. Select the first option and select the name of your queue from the list.
4. Click show advanced options and select DeadLetter for queue type.
5. Then click add condition. You should see a true false decision tree as above. While writing the condition you can refer to the properties of the message we received from the dead-letter queue. In the example above, we make a decision based on whether the message contains a piece of text as "taskType":"certainTaskType", a JSON snippet.
6. If the condition is true, we'll call another Azure function and send some information about our message. Assume that it will take some remedying action. If the condition is false, then we send the dev team an email using a logic apps connector such as Office 365 or Gmail.

## Next steps

If you're interested in going further, here is what we could add to this setup to make it even sweeter:

* Automatically deploy functions from Github, Bitbucket or Visual Studio Team Services.
* Add API management in front of our HTTP triggered function to better manage access to our endpoint.

If you have any questions, let me know in the issues.
