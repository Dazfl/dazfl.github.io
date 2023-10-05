---
layout: post
title: How I implement Commands and Queries
excerpt_separator: <!--more-->
---

When building applications, I like to use the CQRS and Mediator pattern. This is how I typically set up my Commands and Queries using the MediatR library.<!--more-->

# CQRS and MediatR

## The Basics

If I tried to explain the concepts of CQRS and the Mediator pattern, I wouldn't be able to do them justice, so I'll just provide a link to a YouTube video that explains it very well (BTW, Tim Corey is a great teacher). This video also explains the [MediatR](https://github.com/jbogard/MediatR "MediatR") library.

- [Intro to MediatR - Implementing CQRS and Mediator Patterns](https://www.google.com.au/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiexKb2iNyBAxVGs1YBHZX3DesQwqsBegQIUxAB&url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DyozD5Tnd8nw&usg=AOvVaw0RJBEHe526MVzquNTEuQFw&opi=89978449 "Intro to MediatR - Implementing CQRS and Mediator Patterns")

## The Code

The examples below are how I typically set up a Command and Query.  I find doing it this way makes maintenance and debugging very easy. It keeps everything together very nicely, while still giving you the flexibility to move logic into seperate files.

The examples below return the custom **Response** object I use for my projects (details of how I implement that are [here](https://dazfl.github.io/2023/09/09/simple-response-class.html "Response Class")).


### Command
```csharp
public static class UpdateOrder
{
    // Command request
    public record Command(Order OrderDetails) : IRequest<Response>;

    // Command handler
    internal sealed CommandHandler : IRequestHandler<Command, Response>
    {
        private readonly ILogger<CommandHandler> _logger;

        public CommandHandler(ILogger<CommandHandler> logger, ...)
        {
            // Constructor ... dependency injection set up can go here
            _logger = logger;
        }

        public async Task<Response> Handle(Command request, CancellationToken cancellationToken)
        {
            // Logic to handle updating an Order ...

            if (successful)
                return Response.Success;
            
            return Response.Failure("Could not update Order.");
        }
    }
}
```
### Query
```csharp
public static class GetOrders
{
    // Query request
    public record Query(int CustomerId) : IRequest<Response<ICollection<Order>>>;

    // Query handler
    internal sealed QueryHandler : IRequestHandler<Query, Response<ICollection<Order>>>
    {
        private readonly ILogger<QueryHandler> _logger;

        public QueryHandler(ILogger<QueryHandler> logger, ...)
        {
            // Constructor ... dependency injection set up can go here
        }

        public async Task<Response<ICollection<Order>>> Handle(Query request, CancellationToken cancellationToken)
        {
            // Logic to retrieve a list of Orders
            var orders = _dbContext.Orders
                .AsNoTracking()
                .Where(x => x.CustomerId == request.CustomerId)
                .ToList();

            return Response.Success
                .WithResults<ICollection<Order>>(orders);
        }
    }
}
```
### Calling the Command and/or Query
I usually do this from a controller/minimal API. This way I can keep the endpoints very thin.
```csharp
[HttpGet("Orders/{customerId}")]
public async Task<IActionResult> GetOrders(int customerId, CancellationToken cancellationToken)
{
    var queryResponse = await _bus.Send(new GetOrders.Query(customerId), cancellationToken);
    if (queryResponse.IsSuccessful)
        return Ok(queryResponse);

    return BadRequest(queryResponse);
}

[HttpPut("Orders")]
public async Task<Response> UpdateOrders(Order orderToUpdate, CancellationToken cancellationToken)
{
    return await _bus.Send(new UpdateOrder.Command(orderToUpdate), cancellationToken);
}
```

## Additional Comments

Wrapping the request object, the handler, and any other relevant objects, in a single _static_ class makes life a lot easier for debugging and maintaining the logic for each Command and Query.  You always have the option to split the logic into separate files if things become to unwieldy.

You can also take it a step futher and include validation in the same static class, using a third-party library such as [Fluent Validation](https://github.com/FluentValidation/FluentValidation "Fluent Validation") (such an amazing validation library), and [MediatR pipeline behaviours](https://github.com/jbogard/MediatR/wiki/Behaviors "MediatR pipeline behaviours"). I'll do another post to demonstrate how this gets set up.

Another possibility is you can define a custom response object in the main Static class.  For example,
```csharp
public static class GetOrders
{
    // Query request
    public record Query(int CustomerId) : IRequest<Response<ICollection<OrderSummary>>>;

    // Query handler
    internal sealed QueryHandler ...

    // Query response
    public record OrderSummary()
    {
        // Properties go here ...
    }
}
```
