# A serverless approach to async task management
## Overview
Here we present an easy way to:
* Create an API that accepts requests for task completion.
* A queue mechanism to decouple the app that requests the task and the app that completes the task.
* Create a way to manage tasks which could not be completed.

...using serverless technologies on Azure.

This approach can be applied to any scenario where a requested task is completed asynchrounously such as:

* Image processing
* File conversion
* Invoice generation
* Sending a non-transactional sms/email

...or any other task which doesn’t have to be completed at the time it was requested.

Here is an overview of what we’ll be covering:
![alt text](https://github.com/emrekenci/azure-serverless-async-task/raw/master/images/A%20serverless%20approach%20to%20async%20task%20management.png "A serverless approach to async task management")

The interesting thing about this approach isn't the architecture itself but how little **_code & effort_**  it takes to implement this on Azure. The rest of the article describes the implementation details.

## Implementation guide

