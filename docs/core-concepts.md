# Core Concepts

This document covers the fundamental types in Lua-CSharp: `LuaValue`, `LuaTable`, and `LuaFunction`.

## LuaValue

`LuaValue` is a discriminated union struct that represents all value types in Lua. It's designed for zero-allocation for primitive types.

### Value Types

| Lua Type | C# Type | Description |
|----------|---------|-------------|
| `nil` | `LuaValue.Nil` | Nil/null value |
| `boolean` | `bool` | Boolean values |
| `number` | `double`, `float`, `int`, `long` | Numeric values |
| `string` | `string` | Text strings |
| `table` | `LuaTable` | Lua tables |
| `function` | `LuaFunction` | Lua functions |
| `thread` | `LuaState` | Coroutines |
| `userdata` | `ILuaUserData` | Custom user data |
| light userdata | `object` | Raw pointer-like data |

### Creating LuaValue

```csharp
// Implicit conversion from C# types
LuaValue nilValue = LuaValue.Nil;
LuaValue boolValue = true;
LuaValue numberValue = 42.5;
LuaValue stringValue = "hello";
LuaValue tableValue = new LuaTable();
LuaValue functionValue = new LuaFunction((ctx, ct) => new ValueTask<int>(0));
```

### Reading LuaValue

Use `TryRead<T>` for safe reading or `Read<T>` for throwing conversion:

```csharp
var results = await state.DoStringAsync("return 42");

// Safe reading
if (results[0].TryRead<double>(out var value))
{
    Console.WriteLine(value);  // 42
}

// Throws on failure
var value = results[0].Read<double>();
```

### Supported Types for Reading

```csharp
// Numbers
results[0].Read<double>()
results[0].Read<float>()
results[0].Read<int>()
results[0].Read<long>()
results[0].Read<uint>()
results[0].Read<ulong>()

// Other types
results[0].Read<bool>()
results[0].Read<string>()
results[0].Read<LuaTable>()
results[0].Read<LuaFunction>()
results[0].Read<LuaState>()  // for coroutines
results[0].Read<ILuaUserData>()  // for userdata
```

### Type Checking

```csharp
var value = results[0];

// Check type
if (value.Type == LuaValueType.Number)
{
    var num = value.Read<double>();
}

// Common type checks
bool isNil = value.Type == LuaValueType.Nil;
bool isTable = value.Type == LuaValueType.Table;
bool isFunction = value.Type == LuaValueType.Function;
```

### Type Conversion

```csharp
// Convert to boolean
bool b = value.ToBoolean();  // false for nil, true for everything else

// Convert to string
string s = value.ToString();

// Get type name
string typeName = value.TypeToString();  // "number", "string", etc.
```

### Implicit Conversion

Lua-CSharp provides implicit conversion from C# types:

```csharp
// All of these create LuaValue automatically
LuaValue v1 = 42;           // int -> double
LuaValue v2 = 3.14;         // double
LuaValue v3 = "text";       // string
LuaValue v4 = true;        // bool
LuaValue v5 = new LuaTable();
LuaValue v6 = new LuaFunction(...);
```

### FromObject

Use `LuaValue.FromObject` for complex objects:

```csharp
// Convert C# objects to LuaValue
var result = LuaValue.FromObject(someObject);

// Special handling:
// - null -> LuaValue.Nil
// - LuaValue -> unchanged
// - bool, double, string, etc. -> converted
// - ILuaUserData -> userdata
// - Other objects -> light userdata
```

### Equality and Hashing

```csharp
var a = LuaValue.FromObject(1);
var b = LuaValue.FromObject(1);
var c = LuaValue.FromObject(2);

bool equal = a == b;     // true
bool notEqual = a != c;  // true

// For use in dictionaries/tables
bool dictEqual = a.EqualsForDict(b);
```

## LuaTable

`LuaTable` represents Lua tables - the primary data structure in Lua.

### Creating Tables

```csharp
// Create empty table
var table = new LuaTable();

// Create with capacity hints
var tableWithArray = new LuaTable(100);           // 100 array slots
var tableWithHash = new LuaTable(0, 50);          // 50 hash slots
var tableWithBoth = new LuaTable(100, 50);       // 100 array, 50 hash
```

### Indexing

```csharp
var table = new LuaTable();

// Set values (1-indexed arrays)
table[1] = "first";
table[2] = "second";
table[3] = "third";

// String keys
table["name"] = "Alice";
table["age"] = 30;

// Mixed keys
table["key"] = "value";
table[10] = 100;

// Using LuaValue as key
table[LuaValue.FromObject("key")] = "value";
```

### Reading Values

```csharp
var table = results[0].Read<LuaTable>();

// Read by index (1-based)
var first = table[1];
var second = table[2];

// Read by string key
var name = table["name"];

// Read with default for missing keys
var missing = table["nonexistent"];  // Returns LuaValue.Nil
```

### Iteration

```csharp
var table = results[0].Read<LuaTable>();

// Iterate all key-value pairs
foreach (var (key, value) in table)
{
    Console.WriteLine($"{key} = {value}");
}

// Iterate with LuaValue
foreach (var kvp in table.AsEnumerable())
{
    Console.WriteLine(kvp.Key);
    Console.WriteLine(kvp.Value);
}
```

### Table Operations

```csharp
// Get array portion as span (1-indexed)
var array = table.GetArraySpan();

// Get count
int count = table.Count;

// Check if empty
bool isEmpty = table.Count == 0;

// Ensure capacity
table.EnsureArrayCapacity(100);
```

### Metatable

```csharp
var table = new LuaTable();
var metatable = new LuaTable();

metatable["__add"] = new LuaFunction(...);
table.Metatable = metatable;
```

## LuaFunction

`LuaFunction` represents both Lua functions and C# functions callable from Lua.

### Creating C# Functions

```csharp
// Simple function
var func = new LuaFunction((context, ct) =>
{
    // Get arguments
    var arg0 = context.GetArgument<double>(0);
    var arg1 = context.GetArgument<double>(1);
    
    // Return values
    context.Return(arg0 + arg1);
    return new ValueTask<int>(1);  // 1 return value
});

// Function with named parameter
var greetFunc = new LuaFunction("greet", (context, ct) =>
{
    var name = context.GetArgument<string>(0);
    context.State.Stack.Push($"Hello, {name}!");
    return new ValueTask<int>(1);
});
```

### LuaFunctionExecutionContext

The context provides access to the function's execution state:

```csharp
new LuaFunction((context, ct) =>
{
    // Access state
    LuaState state = context.State;
    
    // Get arguments
    int argCount = context.ArgumentCount;
    var firstArg = context.GetArgument<string>(0);
    
    // Return values (multiple ways)
    context.Return("value1", "value2");      // Multiple values
    // or
    context.State.Stack.Push("value1");
    context.State.Stack.Push("value2");
    
    return new ValueTask<int>(2);  // Number of return values
});
```

### Calling Functions

```csharp
// Call with array
var results = await state.CallAsync(function, [arg1, arg2, arg3]);

// Call with explicit argument count
var results2 = await state.CallAsync(function, 2);  // 2 args from stack

// Low-level stack-based calls
state.Stack.Push(function);
state.Stack.Push(arg1);
state.Stack.Push(arg2);
var returnCount = await state.CallAsync(functionIndex: basePos, returnBase: basePos);
```

### Function Return Values

```csharp
// Single return value
context.Return(value);  // or context.Return(singleValue);

// Multiple return values
context.Return(value1, value2, value3);
// or
state.Stack.Push(value1);
state.Stack.Push(value2);
return new ValueTask<int>(3);  // Return count

// Return nil
context.Return();  // Returns nil
// or
return new ValueTask<int>(0);  // No values = nil
```

### Named Functions

```csharp
// Function with name (useful for debugging)
var myFunc = new LuaFunction("myFunction", async (context, ct) =>
{
    // Function name shows in stack traces
    return new ValueTask<int>(0);
});

state.Environment["myFunction"] = myFunc;
```

### Async Functions

Functions can be async for non-blocking operations:

```csharp
state.Environment["delay"] = new LuaFunction(async (context, ct) =>
{
    var seconds = context.GetArgument<double>(0);
    await Task.Delay(TimeSpan.FromSeconds(seconds), ct);
    context.Return();
    return new ValueTask<int>(0);
});

// Lua: delay(1)  -- waits 1 second
```

## Working with All Types Together

### Complete Example

```csharp
var state = LuaState.Create();

// Create various values
state.Environment["number"] = 42.5;
state.Environment["text"] = "hello";
state.Environment["flag"] = true;
state.Environment["data"] = new LuaTable();

// Execute script
var results = await state.DoStringAsync(@"
    local x = number
    local y = text
    local z = flag
    local t = data
    
    -- Function returning multiple values
    return x, y, z, 'done'
");

// Read results
double num = results[0].Read<double>();
string txt = results[1].Read<bool>() ? "yes" : "no";
string status = results[3].Read<string>();
```

### Type Mapping Table

| Lua Value | C# Read Method |
|-----------|----------------|
| `nil` | `results[0].Type == LuaValueType.Nil` |
| `true`/`false` | `results[0].Read<bool>()` |
| `42` | `results[0].Read<double>()` |
| `"text"` | `results[0].Read<string>()` |
| `{1, 2, 3}` | `results[0].Read<LuaTable>()` |
| `function() end` | `results[0].Read<LuaFunction>()` |
| `coroutine.create(...)` | `results[0].Read<LuaState>()` |

## Best Practices

### Memory Efficiency

```csharp
// Prefer TryRead to avoid allocations from Read throwing
if (value.TryRead<double>(out var d))
{
    // Use d
}

// Use span-based access when possible
var span = table.GetArraySpan();
foreach (var item in span)
{
    // Process
}
```

### Type Safety

```csharp
// Always check types before reading
if (value.Type == LuaValueType.Number)
{
    var n = value.Read<double>();
}

// Or use TryRead which is safer
value.TryRead<double>(out var number);
value.TryRead<string>(out var str);
```

### Stack Management

```csharp
// When doing low-level operations, manage the stack carefully
var basePos = state.Stack.Count;
state.Push(function);
state.Push(arg1);
// ... more arguments ...

var resultsCount = await state.CallAsync(basePos, basePos);

using (var reader = state.ReadStack(resultsCount))
{
    var results = reader.AsSpan();
    // Process results
}
```

## Next Steps

- [API Reference](api-reference.md) - Complete API documentation
- [Advanced Topics](advanced-topics.md) - Coroutines, metamethods, async
- [Standard Libraries](standard-libraries.md) - Available standard libraries