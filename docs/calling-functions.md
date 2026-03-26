# Calling Functions Between C# and Lua

This guide covers how to call C# functions from Lua and how to call Lua functions from C#.

## Calling C# Functions from Lua

There are two approaches to expose C# functions to Lua:

### 1. Using the Source Generator (Recommended)

The easiest way to expose C# classes to Lua is using the `[LuaObject]` attribute with the source generator.

#### Mark Your Class

```csharp
using Lua;

[LuaObject]
public partial class MyClass
{
    [LuaMember]
    public int Add(int a, int b) => a + b;

    [LuaMember]
    public string Greet(string name) => $"Hello, {name}!";

    [LuaMember]
    public static string StaticMethod() => "Static!";
}
```

#### Use from Lua

```lua
local obj = MyClass()
print(obj:Add(1, 2))           -- 3
print(obj:Greet("World"))      -- Hello, World!
print(MyClass.StaticMethod())  -- Static!
```

### 2. Using LuaFunction Directly

For simple function exposure without the source generator, use `LuaFunction`:

#### Basic Function

```csharp
var state = LuaState.Create();

// Simple synchronous function
state.Environment["greet"] = new LuaFunction("greet", (context, ct) =>
{
    string name = context.GetArgument<string>(0);
    return context.Return($"Hello, {name}!");
});
```

```lua
print(greet("World"))  -- Hello, World!
```

#### Function with Multiple Returns

```csharp
state.Environment["divmod"] = new LuaFunction("divmod", (context, ct) =>
{
    int a = context.GetArgument<int>(0);
    int b = context.GetArgument<int>(1);
    return context.Return(a / b, a % b);
});
```

```lua
local q, r = divmod(10, 3)
print(q, r)  -- 3 1
```

#### Accessing Arguments

```csharp
new LuaFunction("myFunc", (context, ct) =>
{
    // Get argument count
    int count = context.ArgumentCount;

    // Get typed argument
    int num = context.GetArgument<int>(0);
    string str = context.GetArgument<string>(1);
    bool flag = context.GetArgument<bool>(2);

    // Get as LuaValue
    LuaValue any = context.GetArgument(0);

    // Access state
    LuaState state = context.State;

    return context.Return(LuaValue.Nil);
});
```

#### Returning Values

```csharp
// Single return
return context.Return(42);

// Multiple returns
return context.Return(1, 2, 3);

// Return nothing (nil)
return context.Return();

// Return LuaValue
return context.Return(new LuaValue("hello"));
```

### 3. Async Functions

```csharp
state.Environment["asyncGreet"] = new LuaFunction("asyncGreet", async (context, ct) =>
{
    string name = context.GetArgument<string>(0);
    await Task.Delay(100, ct);  // Simulate async work
    return context.Return($"Hello, {name}!");
});
```

```lua
print(asyncGreet("World"))  -- Hello, World!
```

### 4. Exposing C# Objects

#### Automatic Conversion

C# objects are automatically converted to userdata:

```csharp
public class MyService
{
    public string Name { get; set; }
    public int Calculate(int x) => x * 2;
}

var service = new MyService { Name = "MyService" };
state.Environment["service"] = service;
```

```lua
print(service.Name)           -- MyService
print(service:Calculate(21))  -- 42
```

#### With Source Generator

```csharp
using Lua;

[LuaObject]
public partial class Calculator
{
    int _value;

    [LuaMember("create")]
    public static Calculator Create(int initialValue)
        => new() { _value = initialValue };

    [LuaMember]
    public int Add(int amount) => _value += amount;

    [LuaMember]
    public int GetValue() => _value;
}
```

```lua
local calc = Calculator.create(10)
calc:Add(5)
print(calc:GetValue())  -- 15
```

---

## Calling Lua Functions from C#

### 1. Getting a Lua Function

```csharp
// From global environment
LuaFunction? func = state.Environment["myFunction"].TryRead<LuaFunction>(out var f) ? f : null;

// From a table
LuaFunction? func = state.Environment["math"]["floor"].TryRead<LuaFunction>(out var f) ? f : null;
```

### 2. Calling with CallAsync

```csharp
LuaFunction? func = state.Environment["add"].TryRead<LuaFunction>(out var f) ? f : null;

if (func != null)
{
    var results = await state.CallAsync(func, [1, 2]);
    int sum = results[0].Read<int>();  // 3
}
```

### 3. Calling with RunAsync

```csharp
// Push function and arguments to stack, then run
state.Push(func);
state.Push(1);
state.Push(2);

int results = await state.RunAsync(func, 3);  // 3 arguments
```

### 4. Calling with DoString

```csharp
// Define function in Lua, then call it
await state.DoStringAsync(@"
    function add(a, b)
        return a + b
    end
");

var results = await state.DoStringAsync("return add(1, 2)");
int sum = results[0].Read<int>();  // 3
```

### 5. Callback Pattern

Expose a C# callback that Lua can call:

```csharp
state.Environment["setCallback"] = new LuaFunction("setCallback", (context, ct) =>
{
    var callback = context.GetArgument<LuaFunction>(0);
    state.Environment["myCallback"] = callback;
    return context.Return();
});
```

```lua
setCallback(function(x)
    print("Callback called with:", x)
    return x * 2
end)

myCallback(21)  -- Callback called with: 21
```

### 6. Passing C# Functions as Callbacks

```csharp
// Create a LuaFunction that wraps a C# Action
Action<int> csharpCallback = x => Console.WriteLine($"C# callback: {x}");

state.Environment["callCpp"] = new LuaFunction("callCpp", (context, ct) =>
{
    var luaCallback = context.GetArgument<LuaFunction>(0);
    luaCallback.Call(42);  // Call the Lua function from C#
    return context.Return();
});
```

```lua
callCpp(function(x)
    print("Lua received:", x)
end)
```

---

## Complete Examples

### Example 1: C# Math Library Wrapper

```csharp
using Lua;

[LuaObject]
public partial class MathUtils
{
    [LuaMember("add")]
    public static int Add(int a, int b) => a + b;

    [LuaMember("sub")]
    public static int Sub(int a, int b) => a - b;

    [LuaMember("mul")]
    public static int Mul(int a, int b) => a * b;

    [LuaMember("div")]
    public static double Div(double a, double b) => a / b;

    [LuaMember("pow")]
    public static double Pow(double a, double b) => Math.Pow(a, b);
}
```

```lua
local Math = MathUtils

print(Math.add(2, 3))      -- 5
print(Math.mul(4, 5))      -- 20
print(Math.pow(2, 8))      -- 256.0
```

### Example 2: Lua Callback for Events

```csharp
var state = LuaState.Create();
state.OpenBasicLibrary();

// Event system
var callbacks = new List<LuaFunction>();

state.Environment["registerCallback"] = new LuaFunction("register", (context, ct) =>
{
    var callback = context.GetArgument<LuaFunction>(0);
    callbacks.Add(callback);
    return context.Return(callbacks.Count);
});

state.Environment["triggerEvent"] = new LuaFunction("trigger", async (context, ct) =>
{
    string message = context.GetArgument<string>(0);
    foreach (var callback in callbacks)
    {
        await state.CallAsync(callback, [message]);
    }
    return context.Return();
});
```

```lua
registerCallback(function(msg)
    print("Event received:", msg)
end)

triggerEvent("Hello!")  -- Event received: Hello!
```

### Example 3: C# Calling Lua Iterator

```csharp
var state = LuaState.Create();
await state.DoStringAsync(@"
    function range(n)
        local i = 0
        return function()
            i = i + 1
            if i <= n then return i end
        end
    end
");

var rangeFunc = state.Environment["range"].TryRead<LuaFunction>(out var f) ? f : null;
if (rangeFunc != null)
{
    var results = await state.CallAsync(rangeFunc, [5]);
    var iterator = results[0].Read<LuaFunction>();
    
    // Call iterator repeatedly
    while (true)
    {
        var iterResults = await state.CallAsync(iterator);
        if (iterResults.Length == 0 || iterResults[0].Type == LuaValueType.Nil)
            break;
        Console.WriteLine(iterResults[0].Read<int>());  // Prints 1, 2, 3, 4, 5
    }
}
```

### Example 4: Async Lua Function from C#

```csharp
var state = LuaState.Create();
await state.DoStringAsync(@"
    function fetchData(id)
        return 'Data-' .. id
    end
    
    async function fetchAsync(id)
        return 'Async-' .. id
    end
");

// Call sync function
var syncResult = await state.DoStringAsync("return fetchData(123)");
Console.WriteLine(syncResult[0].Read<string>());  // Data-123

// Call async function
var asyncResult = await state.DoStringAsync("return fetchAsync(456)");
Console.WriteLine(asyncResult[0].Read<string>());  // Async-456
```

---

## Type Conversion Reference

### C# to Lua

| C# Type | Lua Type |
|---------|----------|
| `null` | nil |
| `bool` | boolean |
| `int`, `long`, `float`, `double` | number |
| `string` | string |
| `LuaValue` | (as-is) |
| `LuaTable` | table |
| `LuaFunction` | function |
| `[LuaObject]` | userdata |
| `ILuaUserData` | userdata |

### Lua to C#

| Lua Type | C# Type |
|----------|---------|
| nil | `null` |
| boolean | `bool` |
| number | `int`, `long`, `double` |
| string | `string` |
| table | `LuaTable` |
| function | `LuaFunction` |
| userdata | `ILuaUserData`, `[LuaObject]` type |

---

## Error Handling

### Handling Lua Errors in C#

```csharp
try
{
    var results = await state.DoStringAsync("error('something wrong')");
}
catch (LuaRuntimeException ex)
{
    Console.WriteLine(ex.Message);  // Lua error: something wrong
    Console.WriteLine(ex.LuaTraceback);  // Stack trace
}
```

### Throwing Errors from C# Functions

```csharp
state.Environment["validate"] = new LuaFunction("validate", (context, ct) =>
{
    int value = context.GetArgument<int>(0);
    if (value < 0)
    {
        throw new LuaRuntimeException("Value must be non-negative", context.State);
    }
    return context.Return(value);
});
```

```lua
validate(-1)  -- Error: Value must be non-negative
```

---

## Best Practices

1. **Use Source Generator for complex classes** - Reduces boilerplate
2. **Name functions clearly** - Use descriptive names in both C# and Lua
3. **Handle async properly** - Use `async`/`await` for non-blocking operations
4. **Validate input** - Check arguments in C# functions
5. **Return meaningful errors** - Use `LuaRuntimeException` for errors
6. **Avoid heavy operations in callbacks** - Offload to separate tasks

---

## Related Documentation

- [Source Generator](source-generator.md) - Detailed source generator docs
- [API Reference](api-reference.md) - Complete API
- [Advanced Topics](advanced-topics.md) - Metamethods, coroutines
- [Exception Handling](exception-handling.md) - Error handling
