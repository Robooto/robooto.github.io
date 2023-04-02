---
layout: post
title: "Async and await gotchas"
---

# [](#C#-Async-and-await-gotchas) C# Async and await gotchas

There are a couple gotchas with async/await in c# that usually present themselves when you first get started with async/await. The biggest is making sure you use async await every chance you get, and the other is how to deal with async functions that don't have a return.


### [](#async-sync-mix-and-match)Async all the way through

Mixing async and sync is another gotcha developers can fall into when they first start using async await.  The place I've seen it the most is http calls.  First you use `await` when you get your response, but you need to use `await` again when you get the content.  This second `await` I've often seen replaced by `.Result`.  

```c#
public async Task<string> GetSomeData()
{
    var client = new HttpClient();
    var response = await client.GetAsync("https://www.google.com");
    response.EnsureSuccessStatusCode();
    // This is done synchronously due .Result
    var content = response.Content.ReadAsStringAsync().Result;
    return content;
}
```

Using `.Result` here runs this code synchronously while the first request runs async this negates the async thread optimization and will require multiple threads when normally one could do all the work.  In a large enough case, this could lead to thread-pool starvation.  To address this just replace the `.Result` with another `await`.

```c#
public async Task<string> GetSomeData()
{
    var client = new HttpClient();
    var response = await client.GetAsync("https://www.google.com");
    response.EnsureSuccessStatusCode();
    var content = await response.Content.ReadAsStringAsync();
    return content;
}
```
Now we've made our compiler happy, we're using fewer threads, and have a consistent style in our code.

### [](#async-void)Async Void

There are times when you have an async method that doesn't return anything `void`.  If you're refactoring non-async code to async you'd probably just continue to keep the return type of `void`.   The compiler doesn't yell at you if you do this but can be very dangerous for your app.

In this contrived example, we have a function where we don't care what happens after we call it but what happens if it throws and exception?

```c#
using fun;

var testing = new TestAsnyc();
Console.WriteLine("Hello, World!");

await MyAsyncVoidFunction();
async Task MyAsyncVoidFunction()
{
    try
    {
        testing.MyAsyncFunction(true);
    }
    catch (Exception ex)
    {
        // Handle exception here
        Console.WriteLine("Exception caught in MyAsyncFunction");
    }
}
public class TestAsnyc
{
    public async void MyAsyncFunction(bool throwException = false)
    {
        await Task.Delay(1000);
        Console.WriteLine("A thing happened that I don't care about.");
        if (throwException)
            throw new Exception("Exception from MyAsyncFunction");
    }
}
```

```
Hello, World!

Process finished with exit code 0.
```
Since we aren't awaiting this method, we don't get the exception in our code because exceptions aren't captured by the calling method.  In some cases, this unhandled exception could lead to our app crashing.

How do we address this?  Use a Task instead of void.

```c#
using fun;

var testing = new TestAsnyc();
Console.WriteLine("Hello, World!");

await MyAsyncVoidFunction();
async Task MyAsyncVoidFunction()
{
    try
    {
        await testing.MyAsyncFunction(true);
    }
    catch (Exception ex)
    {
        // Handle exception here
        Console.WriteLine("Exception caught in MyAsyncFunction");
    }
}
public class TestAsnyc
{
    public async Task MyAsyncFunction(bool throwException = false)
    {
        await Task.Delay(1000);
        Console.WriteLine("A thing happened that I don't care about.");
        if (throwException)
            throw new Exception("Exception from MyAsyncFunction");
    }
}
```
Since we're using `Task` and `await` we are now correctly catching the exception, and we're safe from unhandled exceptions.

```
Hello, World!
A Thing Happened that I don't care about.
Exception caught in MyAsyncFunction

Process finished with exit code 0.
```

The biggest takeaway is never/rarely use `async void` and use `async Task` instead and to always `await` your async calls.

Happy Coding!

[Github with more async guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md)
