---
layout: post
title: Efficient Task Cancellation Using Tokens
---

> SUMMARY
>- Used to signal and handle cancellation of long-running tasks, such as database queries or API calls.
>- Enables **graceful cancellation** for scenarios like timeouts, user-initiated cancellations, or application shutdowns.
>- Best practices include setting **timeouts**, checking `token.IsCancellationRequested` in long-running loops, and always passing the token to methods that support it.

A **cancellation token** is a mechanism used to signal that an operation should be cancelled. It's part of the **cooperative cancellation** model, where a caller can request cancellation, and the callee (the method being called) is responsible for periodically checking whether cancellation has been requested.

---
### **Why use a Cancellation Token in database operations?**

In long-running operations like database queries, cancellations can be helpful in scenarios such as:

1. **Timeout Handling**: If a query takes too long to execute, you may want to cancel it to avoid blocking system resources.
2. **User-Requested Cancellation**: For interactive applications (like web or desktop apps), users might cancel an ongoing operation (e.g., clicking "Cancel" on a form).
3. **Graceful Shutdown**: During application shutdown or restart, in-progress operations can be canceled to release resources quickly and cleanly.
4. **Preventing Overloads**: If a high number of concurrent queries are being processed and a new operation becomes unnecessary (e.g., stale / outdated data queries), you can cancel it to reduce database load.

---
### **How to use a Cancellation Token**

Here’s how to use a `CancellationToken` when calling a database in .NET:

#### **Step 1: Create a `CancellationTokenSource`**
```csharp
var cts = new CancellationTokenSource();
// Optional: specify a timeout period
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)); // Auto-cancel after 10 seconds
CancellationToken token = cts.Token;
```

#### **Step 2: Pass the token to the async database method**
Assuming you're using an ORM, many of their async methods support cancellation tokens:
```csharp
await dbContext.MyTable
               .Where(x => x.Status == "Active")
               .ToListAsync(token); // Pass the token here
```

#### **Step 3: Handle operation cancellation**
You should handle the case where the operation is canceled using a `try-catch` block:
```csharp
try
{
    await dbContext.MyTable.ToListAsync(token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was canceled.");
}
```

---
### **How it works**
1. The `CancellationTokenSource` sends a signal to cancel the operation when `cts.Cancel()` is called or when the timeout expires.
2. The async method periodically checks the `CancellationToken` to see if a cancellation request has been made.
3. If the cancellation is requested, an `OperationCanceledException` is thrown.
4. This exception can be caught to handle the cancellation gracefully (e.g., log it or notify the user).

---
### **Best Practices**

1. **Always check the token inside long-running tasks**: If you're implementing custom logic, periodically check `token.IsCancellationRequested` or call `token.ThrowIfCancellationRequested()` to stop further execution.
    ```csharp
public async Task ProcessDataAsync(CancellationToken token)
{
    for (int i = 0; i < 1000; i++)
    {
        // Periodically check if cancellation is requested
        if (token.IsCancellationRequested)
        {
            Console.WriteLine("Cancellation requested. Exiting...");
            return; // Gracefully exit the method
        }

        // Simulate work
        await Task.Delay(100); // Simulate asynchronous operation
        Console.WriteLine($"Processing item {i}");
    }
}
    ```

2. **Use timeouts**: Timeouts ensure that operations don’t run indefinitely.
    ```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30)); // Cancel after 30 seconds
    ```

3. **Respect cancellation requests**: Always pass the `CancellationToken` to methods that support it. This ensures that the entire call stack respects cancellation.

---
### **Common Use Cases**
- **Web APIs**: Passing a `CancellationToken` from an HTTP request to downstream services ensures that if the client disconnects, server-side operations stop.
- **Background jobs**: In scheduled jobs or long-running tasks, cancellation tokens are used to stop work gracefully during shutdowns or restarts.
- **Database operations**: Prevent unnecessary long-running queries or stale data fetches by cancelling requests when no longer needed.

### **Key Learnings and Feedback**

After sharing this article with a colleague, he shared the following learnings from his past experiences:

1. **Monitoring and False Positives**:

**Cancellation requests** should be handled carefully in monitoring systems. If not managed properly, cancelled tasks can be flagged as **errors** in dashboards, leading to **false positives**. These cancellations should be treated as expected behaviour and not as failures to avoid cluttering monitoring tools with unnecessary alerts.

**Solution**: Add specific logic to distinguish between a **normal cancellation** (e.g., due to user request or timeout) and an actual error. You can log cancellations differently to prevent them from appearing as critical issues in monitoring dashboards.

2. **Handling Cancellation in Commands**:

**Cancellation with commands** can leave the system in an **inconsistent state**, especially when multiple synchronous calls are made to other services. If the cancellation happens in the middle of a process, it might interrupt state changes or cause side effects, resulting in data corruption or incomplete operations.

```csharp
// Step 1: Process the payment
await paymentService.Pay(amount, cancellationToken); // Payment is successful

// Step 2: Simulate cancellation happening between payment and shipping
if (cancellationToken.IsCancellationRequested) 
{
    Console.WriteLine("Cancellation requested after payment, but before shipping.");
    return; // Exit without shipping the product
}

// Step 3: Ship the product (this never gets executed if cancellation happens in between)
await shippingService.Ship(product, cancellationToken);
```

**Solution**: Ensure that your system is designed to **roll back changes** or maintain consistency when a cancellation occurs. Using **transactional** or **compensating actions** can help restore the system to a valid state after a cancellation.