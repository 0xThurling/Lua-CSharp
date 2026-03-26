# Source Generator - LuaObject

The Source Generator provides automatic wrapper code generation for C# classes to be used in Lua. This eliminates boilerplate code when integrating C# objects with Lua.

## Overview

The `[LuaObject]` attribute tells the source generator to generate:
- Implicit conversion to/from `LuaValue`
- Property accessors for Lua
- Method wrappers for Lua
- Metamethod support

## Basic Usage

### 1. Add the Attribute

Mark your class with `[LuaObject]` and make it `partial`:

```csharp
using Lua;

[LuaObject]
public partial class MyClass
{
    // Class implementation
}
```

### 2. Add Members

Add properties and methods with `[LuaMember]`:

```csharp
using Lua;
using System.Numerics;

[LuaObject]
public partial class Vector3
{
    Vector3 _vector;

    [LuaMember("x")]
    public float X
    {
        get => _vector.X;
        set => _vector = _vector with { X = value };
    }

    [LuaMember("y")]
    public float Y
    {
        get => _vector.Y;
        set => _vector = _vector with { Y = value };
    }

    [LuaMember("create")]
    public static Vector3 Create(float x, float y, float z)
    {
        return new Vector3 { _vector = new Vector3(x, y, z) };
    }

    [LuaMember("length")]
    public float Length => _vector.Length();
}
```

### 3. Use in Lua

```lua
local v = Vector3.create(1, 2, 3)
print(v.x, v.y)  -- 1 2
print(v:length())  -- 3.741657...
```

## Attributes

### LuaObjectAttribute

Marks a class for code generation:

```csharp
[LuaObject]                      // Use class name in Lua
[LuaObject("CustomName")]       // Use custom name in Lua
public partial class MyClass { }
```

### LuaMemberAttribute

Exposes a member to Lua:

```csharp
[LuaMember]                    // Use member name as-is
[LuaMember("customName")]      // Use custom name in Lua
public int MyProperty { get; set; }
```

### LuaMetamethodAttribute

Defines a metamethod:

```csharp
[LuaMetamethod(LuaObjectMetamethod.Add)]
public static Vector3 Add(Vector3 a, Vector3 b)
{
    return new Vector3 { _vector = a._vector + b._vector };
}
```

### LuaIgnoreMemberAttribute

Excludes a member from Lua:

```csharp
[LuaIgnoreMember]
public int InternalValue { get; set; }
```

## Complete Example: Vector3

```csharp
using Lua;
using System.Numerics;

[LuaObject]
public partial class LuaVector3
{
    Vector3 vector;

    [LuaMember("x")]
    public float X
    {
        get => vector.X;
        set => vector = vector with { X = value };
    }

    [LuaMember("y")]
    public float Y
    {
        get => vector.Y;
        set => vector = vector with { Y = value };
    }

    [LuaMember("z")]
    public float Z
    {
        get => vector.Z;
        set => vector = vector with { Z = value };
    }

    [LuaMember("create")]
    public static LuaVector3 Create(float x, float y, float z)
    {
        return new LuaVector3 { vector = new Vector3(x, y, z) };
    }

    [LuaMember("normalized")]
    public LuaVector3 Normalized()
    {
        return new LuaVector3 { vector = Vector3.Normalize(vector) };
    }

    [LuaMember("length")]
    public float Length() => vector.Length();

    [LuaMetamethod(LuaObjectMetamethod.Add)]
    public static LuaVector3 Add(LuaVector3 a, LuaVector3 b)
    {
        return new LuaVector3 { vector = a.vector + b.vector };
    }

    [LuaMetamethod(LuaObjectMetamethod.Sub)]
    public static LuaVector3 Sub(LuaVector3 a, LuaVector3 b)
    {
        return new LuaVector3 { vector = a.vector - b.vector };
    }

    [LuaMetamethod(LuaObjectMetamethod.ToString)]
    public override string ToString() => vector.ToString();
}
```

### Lua Usage

```lua
local v1 = Vector3.create(1, 2, 3)
print(v1.x, v1.y, v1.z)  -- 1 2 3

local v2 = v1:normalized()
print(v2.x, v2.y, v2.z)  -- 0.267... 0.534... 0.801...

print(v1 + v2)  -- <1.267..., 2.534..., 3.801...>
print(v1 - v2)  -- <0.732..., 1.465..., 2.198...>
print(tostring(v1))  -- <1, 2, 3>
```

## Supported Types

### Property/Field Types

Must be convertible to/from `LuaValue`:
- Primitives: `bool`, `int`, `long`, `double`, `float`
- Strings
- `LuaValue`, `LuaTable`, `LuaFunction`
- Other `[LuaObject]` types
- `ILuaUserData`

### Method Return Types

Supported return types:
- `void` - returns nil
- Primitive types - converted to Lua
- `Task` / `Task<T>` - async (returns T)
- `ValueTask` / `ValueTask<T>` - async
- Custom types - converted to Lua

### Method Parameters

Parameter types must be convertible from `LuaValue`.

## Metamethods

### Available Metamethods

| Enum Value | Lua Name | Description |
|------------|----------|-------------|
| `Add` | `__add` | Addition (+) |
| `Sub` | `__sub` | Subtraction (-) |
| `Mul` | `__mul` | Multiplication (*) |
| `Div` | `__div` | Division (/) |
| `Mod` | `__mod` | Modulo (%) |
| `Pow` | `__pow` | Exponentiation (^) |
| `Unm` | `__unm` | Unary negation (-) |
| `Concat` | `__concat` | Concatenation (..) |
| `Eq` | `__eq` | Equality (==) |
| `Lt` | `__lt` | Less than (<) |
| `Le` | `__le` | Less or equal (<=) |
| `Index` | `__index` | Index access |
| `NewIndex` | `__newindex` | Index assignment |
| `Call` | `__call` | Callable |
| `ToString` | `__tostring` | String conversion |
| `Len` | `__len` | Length (#) |
| `GC` | `__gc` | Garbage collection |

### Example: Custom Arithmetic

```csharp
[LuaObject]
public partial class Complex
{
    double real, imag;

    [LuaMetamethod(LuaObjectMetamethod.Add)]
    public static Complex Add(Complex a, Complex b)
    {
        return new Complex { real = a.real + b.real, imag = a.imag + b.imag };
    }

    [LuaMetamethod(LuaObjectMetamethod.ToString)]
    public override string ToString() => $"{real} + {imag}i";
}
```

```lua
local a = Complex.new(1, 2)
local b = Complex.new(3, 4)
print(a + b)  -- 4 + 6i
```

## Static Methods

Static methods become Lua functions on the type:

```csharp
[LuaMember("create")]
public static MyClass Create() => new MyClass();
```

```lua
local obj = MyClass.create()
```

## Instance Methods

Instance methods automatically receive `this` as first argument:

```csharp
[LuaMember("doSomething")]
public int DoSomething(int value) => value * 2;
```

```lua
local obj = MyClass.create()
local result = obj:doSomething(5)  -- 10
```

## Async Methods

Async methods are fully supported:

```csharp
[LuaMember("asyncOperation")]
public async Task<string> AsyncOperation(string input)
{
    await Task.Delay(100);
    return input.ToUpper();
}
```

```lua
local result = obj:asyncOperation("hello")  -- "HELLO"
```

## Limitations

### Reserved Metamethods

`__index` and `__newindex` cannot be overridden - they're used internally by the generated code.

### Type Requirements

All types must be convertible to/from `LuaValue`. The source generator will produce a compile error if unsupported types are used.

## Integration

Add the NuGet package:

```powershell
dotnet add package LuaCSharp
```

The source generator is included in the main package.

### Project File

Ensure your project references the source generator:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="LuaCSharp" Version="*" />
  </ItemGroup>
</Project>
```

## Best Practices

### Use Record Types

For simple data classes, use C# records with `with` expressions:

```csharp
[LuaObject]
public partial record Vector3(float X, float Y, float Z)
{
    [LuaMember("create")]
    public static Vector3 Create(float x, float y, float z) 
        => new(x, y, z);
}
```

### Naming Conventions

- Use `[LuaMember("name")]` for Lua-friendly names
- Use PascalCase for C# and snake_case for Lua

### Error Messages

The source generator provides detailed compile-time errors for:
- Unsupported types
- Missing conversions
- Invalid metamethods

## Troubleshooting

### "Type X is not supported"

The type isn't convertible to/from `LuaValue`. Check that all properties and parameters use supported types.

### "Cannot find generated code"

Make sure the class is `partial` and has the `[LuaObject]` attribute.

### Metamethod not working

Ensure the method signature matches the metamethod (e.g., binary operators take two parameters, unary takes one).

## Next Steps

- [API Reference](api-reference.md) - Complete API
- [Advanced Topics](advanced-topics.md) - More about metamethods
- [Platform & Sandboxing](platform-sandboxing.md) - Security features