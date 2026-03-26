# API Reference

This document provides a comprehensive API reference for Lua-CSharp.

## LuaState

The main entry point for Lua execution.

### Creation

```csharp
// Create with default platform
var state = LuaState.Create();

// Create with custom platform
var platform = new LuaPlatform(...);
var state = LuaState.Create(platform);
```

### Execution Methods

#### DoStringAsync

Executes a Lua code string asynchronously.

```csharp
public ValueTask<LuaValue[]> DoStringAsync(
    string code,
    CancellationToken cancellationToken = default)
```

**Example:**
```csharp
var results = await state.DoStringAsync("return 1 + 1");
var sum = results[0].Read<double>();  // 2
```

#### DoFileAsync

Loads and executes a Lua file.

```csharp
public ValueTask<LuaValue[]> DoFileAsync(
    string fileName,
    CancellationToken cancellationToken = default)
```

**Example:**
```csharp
var results = await state.DoFileAsync("script.lua");
```

### Function Calling

#### CallAsync

Calls a Lua function with arguments.

```csharp
public async ValueTask<LuaValue[]> CallAsync(
    LuaValue function,
    LuaValue[]? arguments = null,
    CancellationToken cancellationToken = default)
```

**Example:**
```csharp
var results = await state.CallAsync(addFunction, [1, 2]);
var sum = results[0].Read<double>();  // 3
```

#### RunAsync

Runs a function directly.

```csharp
public ValueTask<int> RunAsync(
    LuaFunction function,
    int argumentCount = 0,
    CancellationToken cancellationToken = default)
```

### Environment and Registry

```csharp
// Access global environment
LuaTable environment = state.Environment;

// Access registry (private global table)
LuaTable registry = state.Registry;

// Access loaded modules
LuaTable loadedModules = state.LoadedModules;

// Access preload modules  
LuaTable preloadModules = state.PreloadModules;
```

### Module Loading

```csharp
// Set custom module loader
state.ModuleLoader = new CustomModuleLoader();

// Get current loader
ILuaModuleLoader? loader = state.ModuleLoader;
```

### Platform

```csharp
// Get or set platform
LuaPlatform platform = state.Platform;
state.Platform = newPlatform;
```

### Coroutine Operations

```csharp
// Create a new coroutine (thread)
LuaState newThread = state.CreateThread();

// Create a coroutine from a function
LuaState coroutine = state.CreateCoroutine(function, isProtectedMode = false);

// Resume a coroutine
var results = await coroutine.ResumeAsync(stack);

// Get coroutine status
LuaThreadStatus status = coroutine.GetStatus();

// Check if can resume
bool canResume = coroutine.CanResume;
```

### Status and Properties

```csharp
// Check if running
bool isRunning = state.IsRunning;

// Check if this is a coroutine
bool isCoroutine = state.IsCoroutine;

// Get current function if coroutine
LuaFunction? coroutineFunction = state.CoroutineFunction;

// Get main thread
LuaState mainThread = state.MainThread;
```

### Stack Operations

```csharp
// Get stack values
ReadOnlySpan<LuaValue> stackValues = state.GetStackValues();

// Get call stack frames
ReadOnlySpan<CallStackFrame> frames = state.GetCallStackFrames();

// Get call stack depth
int frameCount = state.CallStackFrameCount;

// Get current frame
ref readonly CallStackFrame currentFrame = ref state.GetCurrentFrame();
```

### Debugging

```csharp
// Set debug hook
state.SetHook(hookFunction, "lcr", count);

// Clear hook
state.SetHook(null, "");

// Get traceback
Traceback traceback = state.GetTraceback();

// Dump stack (debug)
state.DumpStackValues();
```

### Loading Chunks

```csharp
// Load Lua source
LuaClosure closure = state.Load(chunk, chunkName, environment);

// Load binary chunk (luac)
LuaClosure closure = state.Load(byteChunk, chunkName, "bt", environment);
```

### Disposal

```csharp
// Dispose the state
state.Dispose();

// Note: Cannot dispose while running
```

## LuaValue

Represents any Lua value.

### Properties

```csharp
// Get the type of the value
LuaValueType Type { get; }

// Static nil value
static LuaValue Nil { get; }
```

### Reading Values

```csharp
// Safe read with out parameter
bool TryRead<T>(out T result)

// Throwing read
T Read<T>()

// Unsafe read (faster, no bounds checking)
T UnsafeRead<T>()
```

### Type Checking

```csharp
// Check type
bool isNumber = value.Type == LuaValueType.Number;
bool isString = value.Type == LuaValueType.String;
bool isTable = value.Type == LuaValueType.Table;
bool isNil = value.Type == LuaValueType.Nil;
bool isBool = value.Type == LuaValueType.Boolean;
bool isFunction = value.Type == LuaValueType.Function;
bool isThread = value.Type == LuaValueType.Thread;
```

### Conversion

```csharp
// Convert to boolean
bool b = value.ToBoolean();

// Convert to string
string s = value.ToString();

// Get type name
string typeName = value.TypeToString();
```

### Static Methods

```csharp
// Create from object
static LuaValue FromObject(object obj)

// Create userdata
static LuaValue FromUserData(ILuaUserData? userData)

// Get LuaValueType from Type
static bool TryGetLuaValueType(Type type, out LuaValueType result)

// Get type name from enum
static string ToString(LuaValueType type)
```

## LuaTable

Represents Lua tables (key-value collections).

### Construction

```csharp
// Empty table
var table = new LuaTable();

// With array capacity
var table = new LuaTable(arraySize);

// With array and hash capacity
var table = new LuaTable(arraySize, hashSize);
```

### Indexing

```csharp
// Set value
table[key] = value;
table["key"] = value;
table[1] = "first";

// Get value
LuaValue v = table[key];
LuaValue v = table["key"];
LuaValue v = table[1];
```

### Iteration

```csharp
// As enumerable
foreach (var kvp in table)
{
    LuaValue key = kvp.Key;
    LuaValue value = kvp.Value;
}

// As span (array portion)
Span<LuaValue> array = table.GetArraySpan();
```

### Properties

```csharp
// Get count
int count = table.Count;

// Metatable
LuaTable? Metatable { get; set; }
```

### Methods

```csharp
// Ensure array capacity
void EnsureArrayCapacity(int capacity)

// Try to get value
bool TryGetValue(LuaValue key, out LuaValue value)
```

## LuaFunction

Represents callable functions.

### Construction

```csharp
// Anonymous function
var func = new LuaFunction((context, ct) => new ValueTask<int>(0));

// Named function (better for debugging)
var func = new LuaFunction("myFunction", (context, ct) => new ValueTask<int>(0));

// From delegate
var func = new LuaFunction(SomeDelegateMethod);
```

### LuaFunctionExecutionContext

Provided to functions when called:

```csharp
new LuaFunction((context, ct) =>
{
    // Access state
    LuaState State { get; }
    
    // Argument count
    int ArgumentCount { get; }
    
    // Return frame base
    int ReturnFrameBase { get; }
    
    // Get typed argument
    T GetArgument<T>(int index)
    
    // Get argument as LuaValue
    LuaValue GetArgument(int index)
    
    // Push return values
    void Return(params LuaValue[] values)
    void Return(LuaValue value)
    
    // Get global state
    LuaGlobalState GlobalState { get; }
});
```

## LuaPlatform

Provides environment abstraction for sandboxing.

### Construction

```csharp
var platform = new LuaPlatform(
    FileSystem: fileSystem,
    OsEnvironment: osEnvironment,
    StandardIO: standardIO,
    TimeProvider: timeProvider);
```

### Properties

```csharp
// File system access
IFileSystem FileSystem { get; }

// OS environment (environment variables, etc.)
ILuaOsEnvironment OsEnvironment { get; }

// Standard input/output
ILuaStandardIO StandardIO { get; }

// Time provider
TimeProvider TimeProvider { get; }
```

## Standard Library Extensions

### OpenLibsExtensions

```csharp
// Open all standard libraries
state.OpenStandardLibraries();

// Open individual libraries
state.OpenBasicLibrary();
state.OpenCoroutineLibrary();
state.OpenIOLibrary();
state.OpenMathLibrary();
state.OpenModuleLibrary();
state.OpenOperatingSystemLibrary();
state.OpenStringLibrary();
state.OpenTableLibrary();
state.OpenDebugLibrary();
state.OpenBitwiseLibrary();
```

## Exceptions

### LuaParseException

Thrown during parsing errors.

```csharp
try { }
catch (LuaParseException ex)
{
    string? chunkName = ex.ChunkName;
    SourcePosition position = ex.Position;
    string message = ex.Message;
}
```

### LuaCompileException

Thrown during compilation errors.

```csharp
try { }
catch (LuaCompileException ex)
{
    string chunkName = ex.ChunkName;
    int offset = ex.OffSet;
    SourcePosition position = ex.Position;
    string message = ex.MainMessage;
    string? nearToken = ex.NearToken;
}
```

### LuaRuntimeException

Thrown during runtime errors.

```csharp
try { }
catch (LuaRuntimeException ex)
{
    LuaValue errorObject = ex.ErrorObject;
    Traceback? traceback = ex.LuaTraceback;
    string message = ex.Message;
}
```

### LuaCanceledException

Thrown when operation is cancelled.

```csharp
try { }
catch (LuaCanceledException ex)
{
    CancellationToken ct = ex.CancellationToken;
    Traceback? traceback = ex.LuaTraceback;
}
```

## Low-Level API

### Stack-Based Operations

```csharp
// Push values
void Push(LuaValue value)
void Push(bool value)
void Push(double value)
void Push(string value)

// Pop values
LuaValue Pop()
void Pop(int count)
void PopUntil(int index)

// Get value at index
LuaValue Get(int index)

// Set value at index
void Set(int index, LuaValue value)

// Get stack count
int Count { get; }

// Clear stack
void Clear()
```

### Direct API Operations

```csharp
// Arithmetic
await state.AddAsync(arg1, arg2);
await state.SubAsync(arg1, arg2);
await state.MulAsync(arg1, arg2);
await state.DivAsync(arg1, arg2);
await state.ModAsync(arg1, arg2);

// Comparison
await state.EqualsAsync(arg1, arg2);
await state.LessThanAsync(arg1, arg2);
await state.LessThanOrEqualsAsync(arg1, arg2);

// Table operations
await state.GetTableAsync(table, key);
await state.SetTableAsync(table, key, value);

// Concatenation
await state.ConcatAsync(values);
```

## ILuaModuleLoader

Interface for custom module loading.

```csharp
public interface ILuaModuleLoader
{
    bool Exists(string moduleName);
    ValueTask<LuaModule> LoadAsync(string moduleName, CancellationToken cancellationToken = default);
}

public readonly struct LuaModule
{
    public LuaValue Value { get; }
    public string? Name { get; }
}
```

## CompositeModuleLoader

Combine multiple loaders:

```csharp
state.ModuleLoader = CompositeModuleLoader.Create(
    loader1,
    loader2,
    loader3
);
```