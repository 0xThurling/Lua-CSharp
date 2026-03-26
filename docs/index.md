# Lua-CSharp Documentation

Welcome to the Lua-CSharp documentation. Lua-CSharp is a high-performance Lua interpreter implemented in C# for .NET and Unity.

## Quick Links

- [Getting Started](getting-started.md) - Installation and basic usage
- [Core Concepts](core-concepts.md) - LuaValue, LuaTable, LuaFunction
- [Calling Functions](calling-functions.md) - C# to Lua and Lua to C# function calls
- [API Reference](api-reference.md) - Complete API documentation
- [Standard Libraries](standard-libraries.md) - Lua standard library implementations
- [Advanced Topics](advanced-topics.md) - Async/await, coroutines, metamethods
- [Unity Integration](unity-integration.md) - Unity support and assets
- [Source Generator](source-generator.md) - LuaObject attribute and code generation
- [Platform & Sandboxing](platform-sandboxing.md) - LuaPlatform for sandboxing
- [Exception Handling](exception-handling.md) - Error handling and debugging

## Overview

Lua-CSharp provides:

- **Lua 5.2 Interpreter** - Full Lua 5.2 implementation in C#
- **Async/Await Integration** - Native async execution for non-blocking scripts
- **High Performance** - Optimized for C#-Lua interoperability
- **Unity Support** - Works with both Mono and IL2CPP
- **Source Generator** - Auto-generates wrapper code via `[LuaObject]` attribute

## Features at a Glance

| Feature | Description |
|---------|-------------|
| Script Execution | `DoStringAsync()`, `DoFileAsync()` |
| Value Types | `LuaValue`, `LuaTable`, `LuaFunction` |
| Coroutines | Full coroutine support via `LuaState` |
| Metatables | Custom behavior via metamethods |
| Module Loading | `ILuaModuleLoader` interface |
| Sandboxing | `LuaPlatform` abstraction |

## Requirements

- .NET Standard 2.1 or higher
- .NET 6.0 or .NET 8.0
- Unity 2021.3+ (for Unity integration)

## NuGet Package

```
dotnet add package LuaCSharp
```