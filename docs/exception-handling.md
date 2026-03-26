# Exception Handling

Lua-CSharp provides comprehensive exception handling with detailed error information and stack traces.

## Exception Types

### LuaParseException

Thrown when Lua code has syntax errors during parsing:

```csharp
try
{
    await state.DoStringAsync("if then end");  // Syntax error
}
catch (LuaParseException ex)
{
    Console.WriteLine($"Parse error: {ex.Message}");
    Console.WriteLine($"Position: {ex.Position}");
}
```

**Properties:**
- `ChunkName` - The name of the chunk being parsed
- `Position` - Source position of the error
- `Message` - Error message

### LuaCompileException

Thrown during bytecode compilation:

```csharp
try
{
    await state.DoStringAsync("return table.nosuchfunction()");
}
catch (LuaCompileException ex)
{
    Console.WriteLine($"Compile error in {ex.ChunkName}:{ex.Position.Line}");
    Console.WriteLine(ex.MainMessage);
}
```

**Properties:**
- `ChunkName` - Name of the chunk
- `OffSet` - Offset in the source
- `Position` - Source position
- `MainMessage` - The main error message
- `NearToken` - Token near the error

### LuaRuntimeException

Thrown when Lua code causes a runtime error:

```csharp
try
{
    await state.DoStringAsync("return nil + 1");  -- Runtime error
}
catch (LuaRuntimeException ex)
{
    Console.WriteLine(ex.Message);
    if (ex.LuaTraceback != null)
    {
        Console.WriteLine(ex.LuaTraceback);
    }
}
```

**Properties:**
- `State` - The Lua state at time of error
- `ErrorObject` - The error value
- `LuaTraceback` - Lua stack trace

### LuaCanceledException

Thrown when script execution is cancelled:

```csharp
var cts = new CancellationTokenSource();

try
{
    await state.DoStringAsync("while true do end", cts.Token);
}
catch (LuaCanceledException ex)
{
    Console.WriteLine("Script was cancelled");
    // Access traceback
    var traceback = ex.LuaTraceback;
}
```

### OperationCanceledException

Standard .NET cancellation:

```csharp
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    await state.DoStringAsync(script, cts.Token);
}
catch (OperationCanceledException)
{
    // Can be either LuaCanceledException or standard
}
```

## Error Handling in Lua

### pcall - Protected Call

Catch errors within Lua:

```lua
local ok, err = pcall(function()
    -- This might error
    error("oops")
end)

if not ok then
    print("Error: " .. err)
end
```

### xpcall - Extended Protected Call

With error handler:

```lua
local function myErrorHandler(msg)
    return "Error caught: " .. msg
end

local ok, result = xpcall(function()
    return someFunction()
end, myErrorHandler)
```

### Assert

```lua
local value = assert(mightBeNil, "value should not be nil")
```

## C# Error Handling Patterns

### Basic Try-Catch

```csharp
try
{
    var results = await state.DoStringAsync(luaCode);
}
catch (LuaParseException ex)
{
    Console.WriteLine($"Syntax error: {ex.Message}");
}
catch (LuaCompileException ex)
{
    Console.WriteLine($"Compile error: {ex.Message}");
}
catch (LuaRuntimeException ex)
{
    Console.WriteLine($"Runtime error: {ex.Message}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation cancelled");
}
```

### Detailed Error Information

```csharp
try
{
    await state.DoFileAsync("script.lua");
}
catch (LuaRuntimeException ex)
{
    Console.WriteLine("=== Error ===");
    Console.WriteLine(ex.Message);
    
    if (ex.LuaTraceback != null)
    {
        Console.WriteLine("=== Stack Trace ===");
        Console.WriteLine(ex.LuaTraceback);
    }
}
```

### Getting Stack Trace

```csharp
// From LuaState
var traceback = state.GetTraceback();
Console.WriteLine(traceback);

// From exception
catch (LuaRuntimeException ex)
{
    var tb = ex.LuaTraceback;
    foreach (var frame in tb.Frames)
    {
        Console.WriteLine($"  {frame.Source}:{frame.Line} in {frame.Name}");
    }
}
```

## Custom Error Handling

### Error Function

Set custom error handling:

```csharp
state.Environment["error"] = new LuaFunction((context, ct) =>
{
    var message = context.GetArgument<string>(0);
    var level = context.HasArgument(1) 
        ? context.GetArgument<int>(1) 
        : 1;
    
    // Custom handling
    Console.WriteLine($"Custom error handler: {message}");
    throw new LuaRuntimeException(context.State, message, level);
});
```

### Rethrowing with Context

```csharp
try
{
    await state.DoStringAsync(code);
}
catch (LuaRuntimeException ex)
{
    // Add more context
    throw new LuaRuntimeException(
        state,
        $"Script failed: {ex.Message}",
        ex);
}
```

## Debug Information

### Getting Current Stack

```csharp
// Get call stack frames
var frames = state.GetCallStackFrames();

foreach (var frame in frames)
{
    Console.WriteLine($"Function: {frame.Function.Name}");
    Console.WriteLine($"  Base: {frame.Base}");
    Console.WriteLine($"  Return Base: {frame.ReturnBase}");
}
```

### Setting Debug Hooks

```csharp
var debugHook = new LuaFunction((context, ct) =>
{
    var eventType = context.State.IsInHook ? "line" : "call";
    Console.WriteLine($"Debug: {eventType}");
    return new ValueTask<int>(0);
});

state.SetHook(debugHook, "lcr");  // line, call, return hooks
```

## Best Practices

### Always Catch Specific Types

```csharp
// Good
try { }
catch (LuaRuntimeException ex) { }

// Better - catch in order of specificity
try { }
catch (LuaParseException ex) { }
catch (LuaCompileException ex) { }
catch (LuaRuntimeException ex) { }
catch (OperationCanceledException ex) { }
```

### Preserve Stack Information

```csharp
catch (LuaRuntimeException ex)
{
    // Don't forget the traceback
    if (ex.LuaTraceback != null)
    {
        Log.Error(ex.Message + "\n" + ex.LuaTraceback);
    }
}
```

### Clean Up on Error

```csharp
LuaState? state = null;
try
{
    state = LuaState.Create();
    await state.DoStringAsync(code);
}
finally
{
    state?.Dispose();
}
```

### Async Cancellation

```csharp
var cts = new CancellationTokenSource();

try
{
    await state.DoStringAsync(code, cts.Token);
}
catch (LuaCanceledException ex)
{
    // Get where it was cancelled
    if (ex.LuaTraceback != null)
    {
        Console.WriteLine($"Cancelled at: {ex.LuaTraceback}");
    }
}
```

## Converting Between C# and Lua Errors

### Throw from C# to Lua

```csharp
state.Environment["fail"] = new LuaFunction((context, ct) =>
{
    throw new LuaRuntimeException(context.State, "Custom error");
});
```

```lua
fail()  -- throws error in Lua
```

### Catch and Convert

```csharp
try
{
    await state.DoStringAsync(luaCode);
}
catch (LuaRuntimeException ex)
{
    // Convert to Lua error for handling
    state.Environment["lastError"] = ex.ErrorObject;
}
```

## Example: Complete Error Handling

```csharp
public class LuaRunner
{
    private readonly LuaState _state;
    
    public LuaRunner()
    {
        _state = LuaState.Create();
    }
    
    public async Task<LuaValue[]> RunScriptAsync(
        string code, 
        CancellationToken ct = default)
    {
        try
        {
            return await _state.DoStringAsync(code, ct);
        }
        catch (LuaParseException ex)
        {
            throw new InvalidOperationException(
                $"Syntax error at line {ex.Position.Line}: {ex.Message}", ex);
        }
        catch (LuaCompileException ex)
        {
            throw new InvalidOperationException(
                $"Compile error at {ex.Position}: {ex.MainMessage}", ex);
        }
        catch (LuaRuntimeException ex)
        {
            var message = ex.LuaTraceback != null
                ? $"{ex.Message}\n{ex.LuaTraceback}"
                : ex.Message;
            throw new InvalidOperationException($"Runtime error: {message}", ex);
        }
        catch (OperationCanceledException ex) when (ex is LuaCanceledException)
        {
            throw new OperationCanceledException(
                "Lua script was cancelled", ex, ct);
        }
    }
    
    public void Dispose()
    {
        _state?.Dispose();
    }
}

// Usage
using var runner = new LuaRunner();
try
{
    var results = await runner.RunScriptAsync("return 1 + 1");
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Script error: {ex.InnerException?.Message}");
}
```

## Next Steps

- [API Reference](api-reference.md) - Complete API documentation
- [Platform & Sandboxing](platform-sandboxing.md) - Security features
- [Getting Started](getting-started.md) - Back to basics