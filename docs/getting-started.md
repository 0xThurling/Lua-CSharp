# Getting Started

This guide will help you get started with Lua-CSharp, from installation to running your first Lua script.

## Installation

### NuGet Package

Lua-CSharp is available as a NuGet package. Install it via:

#### .NET CLI

```powershell
dotnet add package LuaCSharp
```

#### Package Manager

```powershell
Install-Package LuaCSharp
```

#### Package Reference

Add to your `.csproj` file:

```xml
<PackageReference Include="LuaCSharp" Version="*" />
```

### Requirements

- .NET Standard 2.1 or higher
- .NET 6.0 or .NET 8.0
- C# 13 or higher (for primary constructors)

## Your First Script

### Basic Setup

Create a new console application and add Lua-CSharp:

```powershell
dotnet new console -n LuaExample
cd LuaExample
dotnet add package LuaCSharp
```

### Hello World

Here's a simple example that executes a Lua script:

```csharp
using Lua;

// Create a new Lua state
var state = LuaState.Create();

// Execute a simple Lua script
var results = await state.DoStringAsync("print('Hello, World!')");
```

### Basic Arithmetic

Execute arithmetic operations:

```csharp
var state = LuaState.Create();
var results = await state.DoStringAsync("return 1 + 1");

// Access the return value
var value = results[0].Read<double>();  // 2
Console.WriteLine(value);
```

### Multiple Return Values

Lua functions can return multiple values:

```csharp
var state = LuaState.Create();
var results = await state.DoStringAsync("return 1, 2, 3");

// results[0] = 1, results[1] = 2, results[2] = 3
foreach (var result in results)
{
    Console.WriteLine(result);
}
```

## Executing Lua Files

### Loading from File

Use `DoFileAsync` to execute Lua files:

```csharp
var state = LuaState.Create();
var results = await state.DoFileAsync("script.lua");
```

### Writing a Test Script

Create a file called `test.lua`:

```lua
-- test.lua
local name = "Lua-CSharp"
print("Hello from " .. name)

-- Return a value
return 42
```

Execute it:

```csharp
var state = LuaState.Create();
var results = await state.DoFileAsync("test.lua");
var answer = results[0].Read<double>();  // 42
```

## Working with the Global Environment

### Reading Global Variables

Access Lua global variables from C#:

```csharp
var state = LuaState.Create();
await state.DoStringAsync("x = 10");

var x = state.Environment["x"].Read<double>();  // 10
```

### Setting Global Variables

Set globals from C#:

```csharp
var state = LuaState.Create();
state.Environment["name"] = "Alice";
state.Environment["age"] = 30;

var results = await state.DoStringAsync("return name .. ' is ' .. age .. ' years old'");
Console.WriteLine(results[0].Read<string>());  // "Alice is 30 years old"
```

## Working with Lua Tables

### Creating Tables in Lua

```csharp
var state = LuaState.Create();
var results = await state.DoStringAsync("return { a = 1, b = 2 }");

var table = results[0].Read<LuaTable>();
Console.WriteLine(table["a"]);  // 1
Console.WriteLine(table["b"]);  // 2
```

### Creating Tables in C#

```csharp
var state = LuaState.Create();
var table = new LuaTable();
table["key"] = "value";
table[1] = "first";

state.Environment["myTable"] = table;

var results = await state.DoStringAsync("return myTable.key, myTable[1]");
```

## Working with Lua Functions

### Calling Lua Functions from C#

```lua
-- add.lua
local function add(a, b)
    return a + b
end

return add
```

```csharp
var state = LuaState.Create();
var results = await state.DoFileAsync("add.lua");
var addFunction = results[0].Read<LuaFunction>();

// Call the function
var callResults = await state.CallAsync(addFunction, [1, 2]);
var sum = callResults[0].Read<double>();  // 3
```

### Creating C# Functions for Lua

Define functions in C# that Lua can call:

```csharp
var state = LuaState.Create();

// Create a C# function callable from Lua
state.Environment["greet"] = new LuaFunction((context, ct) =>
{
    var name = context.GetArgument<string>(0);
    context.State.Stack.Push($"Hello, {name}!");
    return new(1);  // Return 1 value
});

// Call from Lua
var results = await state.DoStringAsync("return greet('World')");
Console.WriteLine(results[0].Read<string>());  // "Hello, World!"
```

## Using Standard Libraries

Enable Lua's standard libraries:

```csharp
var state = LuaState.Create();
state.OpenStandardLibraries();

// Now you can use math, string, table, etc.
var results = await state.DoStringAsync("return math.pi, string.upper('hello')");
Console.WriteLine(results[0].Read<double>());           // 3.141592653589793
Console.WriteLine(results[1].Read<string>());           // HELLO
```

## Thread Safety Warning

> [!WARNING]
> `LuaState` is **not thread-safe**. Do not access it from multiple threads simultaneously. Each thread should have its own `LuaState` instance.

## Complete Example

Here's a complete example demonstrating multiple features:

```csharp
using Lua;

// Create Lua state
var state = LuaState.Create();

// Add standard libraries
state.OpenStandardLibraries();

// Define a C# function
state.Environment["double"] = new LuaFunction((context, ct) =>
{
    var n = context.GetArgument<double>(0);
    context.Return(n * 2);
    return new(1);
});

// Execute a complex script
var script = @"
    local x = 10
    local y = double(x)
    local text = string.format(' doubled is %d', y)
    return x, y, text
";

var results = await state.DoStringAsync(script);
Console.WriteLine($"{results[0]} doubled is {results[1]}{results[2]}");
// Output: 10 doubled is 20
```

## Next Steps

- [Core Concepts](core-concepts.md) - Learn about LuaValue, LuaTable, and LuaFunction
- [API Reference](api-reference.md) - Complete API documentation
- [Standard Libraries](standard-libraries.md) - Available standard library functions
- [Advanced Topics](advanced-topics.md) - Async/await, coroutines, and metamethods