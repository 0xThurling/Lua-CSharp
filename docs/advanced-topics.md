# Advanced Topics

This document covers advanced features of Lua-CSharp: async/await integration, coroutines, and metamethods.

## Async/Await Integration

Lua-CSharp is designed from the ground up to work seamlessly with C#'s async/await pattern.

### Why Async?

Lua scripts can contain blocking operations (file I/O, network calls, timers). The async API allows these operations to suspend without blocking the calling thread.

### Basic Async Execution

```csharp
var state = LuaState.Create();

// All execution methods are async
var results = await state.DoStringAsync("return 1 + 1");
var fileResults = await state.DoFileAsync("script.lua");
var callResults = await state.CallAsync(function, arguments);
```

### Creating Async C# Functions

Define C# functions that can await:

```csharp
state.Environment["asyncTask"] = new LuaFunction(async (context, ct) =>
{
    // This function can await!
    await Task.Delay(100, ct);
    context.Return("done");
    return new ValueTask<int>(1);
});
```

### Non-Blocking Delays in Lua

```csharp
// Define a wait function in C#
state.Environment["wait"] = new LuaFunction(async (context, ct) =>
{
    var seconds = context.GetArgument<double>(0);
    await Task.Delay(TimeSpan.FromSeconds(seconds), ct);
    return new ValueTask<int>(0);
});
```

Now in Lua:

```lua
print("start")
wait(1)  -- non-blocking wait!
print("after 1 second")
wait(0.5)
print("done")
```

### Flow Control

The script pauses at `wait()` and resumes after the delay:

```
start
[waits 1 second]
after 1 second
[waits 0.5 seconds]
done
```

### Async File Operations

```csharp
// Custom async file loader
state.ModuleLoader = new MyAsyncModuleLoader();

// Files load asynchronously
local module = require("mymodule")
```

### Cancellation

Support for `CancellationToken`:

```csharp
var cts = new CancellationTokenSource();

// Start execution
var task = state.DoStringAsync(luaCode, cts.Token);

// Cancel after 5 seconds
cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    await task;
}
catch (LuaCanceledException)
{
    Console.WriteLine("Script was cancelled");
}
```

Cancel from within Lua:

```csharp
state.Environment["cancelAfter"] = new LuaFunction((context, ct) =>
{
    var seconds = context.GetArgument<double>(0);
    // Schedule cancellation
    Task.Run(async () =>
    {
        await Task.Delay(TimeSpan.FromSeconds(seconds));
        context.State.Platform.TimeProvider.Expire(context.State);
    });
    return new ValueTask<int>(0);
});
```

## Coroutines

Coroutines provide cooperative multitasking within Lua.

### Creating Coroutines

```lua
-- In Lua
local co = coroutine.create(function()
    print("coroutine started")
    coroutine.yield(1)
    print("resumed again")
    return "done"
end)

print(coroutine.status(co))  -- suspended
```

### Resume and Yield

```csharp
var state = LuaState.Create();

// Create coroutine
var results = await state.DoStringAsync(@"
    local co = coroutine.create(function()
        for i = 1, 3 do
            coroutine.yield(i)
        end
        return 'finished'
    end)
    return co
");

var coroutine = results[0].Read<LuaState>();

// Resume the coroutine
var stack = new LuaStack();
for (int i = 0; i < 4; i++)
{
    var count = await coroutine.ResumeAsync(stack);
    if (count > 1)
    {
        Console.WriteLine(stack[1]);  // 1, 2, 3, finished
    }
    stack.Clear();
}
```

### LuaState as Coroutine

In Lua-CSharp, `LuaState` represents both the main state and coroutines:

```csharp
// Create coroutine from Lua function
var results = await state.DoStringAsync(@"
    local function foo()
        coroutine.yield(1)
        return 2
    end
    return coroutine.create(foo)
");

var coroutine = results[0].Read<LuaState>();

// Resume it
var stack = new LuaStack();
await coroutine.ResumeAsync(stack);  // yields 1
await coroutine.ResumeAsync(stack);  // returns 2
```

### Protected Mode

Create coroutines in protected mode (like `coroutine.wrap`):

```csharp
var protectedCoroutine = state.CreateCoroutine(function, isProtectedMode: true);
```

In protected mode, errors don't propagate to the caller.

### Checking Coroutine Status

```csharp
// Get status
LuaThreadStatus status = coroutine.GetStatus();

// Status values:
// Running - currently executing
// Suspended - yielded or not started
// Normal - created but not running
// Dead - finished or errored

bool canResume = coroutine.CanResume;
```

### Complete Coroutine Example

```lua
-- counter.lua
local function counter()
    local i = 0
    while true do
        i = i + 1
        coroutine.yield(i)
    end
end

return coroutine.create(counter)
```

```csharp
var state = LuaState.Create();
var results = await state.DoFileAsync("counter.lua");
var co = results[0].Read<LuaState>();

var stack = new LuaStack();
for (int i = 0; i < 5; i++)
{
    await co.ResumeAsync(stack);
    Console.WriteLine(stack[1]);  // 1, 2, 3, 4, 5
    stack.Clear();
}
```

## Metamethods and Metatables

Metamethods allow customizing behavior of Lua operations.

### Setting Metatable

```csharp
var table = new LuaTable();
var metatable = new LuaTable();

table.Metatable = metatable;
```

### Arithmetic Metamethods

```lua
-- In Lua
local mt = {
    __add = function(a, b) return a.value + b.value end,
    __sub = function(a, b) return a.value - b.value end,
    __mul = function(a, b) return a.value * b.value end,
    __div = function(a, b) return a.value / b.value end,
    __unm = function(a) return -a.value end
}

local a = {value = 10}
local b = {value = 5}
setmetatable(a, mt)
setmetatable(b, mt)

print(a + b)  -- 15
print(a - b)  -- 5
print(a * b)  -- 50
print(a / b)  -- 2
print(-a)     -- -10
```

### Comparison Metamethods

```lua
local mt = {
    __eq = function(a, b) return a.value == b.value end,
    __lt = function(a, b) return a.value < b.value end,
    __le = function(a, b) return a.value <= b.value end
}
```

### Table Access Metamethods

```lua
local mt = {
    __index = function(table, key)
        return "default for " .. tostring(key)
    end,
    __newindex = function(table, key, value)
        print("setting " .. key .. " = " .. tostring(value))
        rawset(table, key, value)
    end
}
```

### String Conversion

```lua
local mt = {
    __tostring = function(t)
        return "MyObject(" .. t.value .. ")"
    end
}
```

### C# Metamethods

Define metamethods in C#:

```csharp
var table = new LuaTable();
var metatable = new LuaTable();

// __add metamethod
metatable[Metamethods.Add] = new LuaFunction((context, ct) =>
{
    var a = context.GetArgument<double>(0);
    var b = context.GetArgument<double>(1);
    context.Return(a + b);
    return new ValueTask<int>(1);
});

table.Metatable = metatable;
state.Environment["table"] = table;
```

```lua
-- Lua
local result = table + 5  -- Uses __add metamethod
```

### Common Metamethods

| Metamethod | Operation |
|------------|-----------|
| `__add` | + |
| `__sub` | - |
| `__mul` | * |
| `__div` | / |
| `__mod` | % |
| `__pow` | ^ |
| `__unm` | unary - |
| `__concat` | .. |
| `__eq` | == |
| `__lt` | < |
| `__le` | <= |
| `__index` | table.key |
| `__newindex` | table.key = value |
| `__call` | table() |
| `__tostring` | tostring() |
| `__len` | # |
| `__gc` | garbage collection |
| `__metatable` | protect metatable |

### Metamethod for Custom Types

Use metamethods with custom C# objects:

```csharp
public class Vector3
{
    public double X, Y, Z;
    
    public static Vector3 operator +(Vector3 a, Vector3 b)
    {
        return new Vector3 { X = a.X + b.X, Y = a.Y + b.Y, Z = a.Z + b.Z };
    }
}
```

```csharp
// Register in Lua
var vec = new LuaVector3(new Vector3(1, 2, 3));
state.Environment["vec"] = vec;
```

## Custom Type Integration

### ILuaUserData

Implement `ILuaUserData` for custom types:

```csharp
public class MyData : ILuaUserData
{
    public LuaTable? Metatable { get; set; }
    public string Name { get; set; }
}
```

### Setting Metatable from C#

```csharp
var userData = new MyData { Name = "test" };
var metatable = new LuaTable();
metatable["__tostring"] = new LuaFunction((ctx, ct) =>
{
    var data = ctx.GetArgument<MyData>(0);
    ctx.Return($"MyData: {data.Name}");
    return new ValueTask<int>(1);
});
userData.Metatable = metatable;
```

## Debug Hooks

Set hooks to monitor execution:

```csharp
// Hook called on line changes, function calls/returns
state.SetHook(hookFunction, "lcr", count);

void SetHook(LuaFunction? hook, string mask, int count = 0)
```

### Hook Masks

- `l` - line hook (each line)
- `c` - call hook (function calls)
- `r` - return hook (function returns)

### Example

```csharp
var hookFunction = new LuaFunction((context, ct) =>
{
    var frame = context.State.GetCurrentFrame();
    // Log hook event
    return new ValueTask<int>(0);
});

state.SetHook(hookFunction, "l", 0);  // line hook
```

## Performance Tips

### Minimize Allocations

```csharp
// Use TryRead instead of Read
if (value.TryRead<double>(out var d)) { }

// Use spans for table iteration
var span = table.GetArraySpan();
```

### Reuse LuaState

Don't create new `LuaState` for each script if possible.

### Async for I/O

Use async methods for file operations to avoid blocking.

## Error Handling

Combine coroutines with protected mode:

```lua
local ok, err = coroutine.resume(co)
if not ok then
    print("Error: " .. err)
end
```

## Next Steps

- [Unity Integration](unity-integration.md) - Unity-specific features
- [Source Generator](source-generator.md) - Automatic C# wrapper generation
- [Platform & Sandboxing](platform-sandboxing.md) - Security and isolation
- [Exception Handling](exception-handling.md) - Error handling patterns