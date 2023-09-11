---
layout: post
title: Simple Response Class
excerpt_separator: <!--more-->
---

# Simple Response Class

This is a simple response class that I use for a consistent a response object to my CQRS commands/queries and APIs.<!--more--> This response object could easily be used for methods that need a consistent reponse, etc (maybe instead of tuples? 😛).  I like using fluent APIs, so I have written it this way.  I'm sure it could be tweaked to be made 'better', but I've found this works for almost all the situations where I've needed a consistent, readable response.

In a file `Response.cs`
```csharp
/// Basic response object
public class Response 
{
    public bool IsSuccessful { get; set; } = true;
    public string Message { get; set; }

    public static Response Success => new();

    public static Response Failure(string message) => new() { IsSuccessful = false, Message = message };
}

/// Response object with additional results property
public class Response<TResult> : Response
{
    public TResult Results { get; set; }
}

/// Extension methods
public static class ResponseExtensions
{
    public static Response<TResult> WithResults<TResult>(this Response response, TResult results)
    {
        return new Response<TResult>() 
        {
            IsSuccessful = response.IsSuccessful,
            Message = response.Message,
            Results = results
        };
    }

    public static Response<TResult> WithNoResults<TResult>(this Response response)
    {
        return new Response<TResult>()
        {
            IsSuccessful = response.IsSuccessful,
            Message = response.Message,
            Results = default
        };
    }
}
```

## Usage

#### Example usage of returning a basic response with no results
```csharp
if (operationIsSuccessful)
    return Response.Success;
else
    return Response.Failure("Oops ... something went wrong!");
```

#### Example usage of returning a response with results
```csharp
// Example results might be a list of objects
IEnumerable<string> results;

if (operationIsSuccessful)
    return Response.Success.WithResults(results);
else
    return Response.Failure("Oops ... something went wrong!")
        .WithNoResults<IEnumerable<string>>();

```

### Example usage of receiving the response
```csharp
var response = GetTheResponseMethod();
if (response.IsSuccessful)
{
    // Do something with the results (if supplied)
    var resultsFromMethod = response.Results;
}
else
{
    // Do something else (maybe log the error)
    logger.LogError(response.Message);
}
```