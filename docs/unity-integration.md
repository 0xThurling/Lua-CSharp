# Unity Integration

Lua-CSharp provides full support for Unity, working with both Mono and IL2CPP.

## Requirements

- Unity 2021.3 or higher
- .NET Standard 2.1 compatibility

## Installation

### Using NuGet for Unity

1. Install [NuGet for Unity](https://github.com/GlitchEnzo/NuGetForUnity)
2. Open NuGet window: `NuGet > Manage NuGet Packages`
3. Search for and install `LuaCSharp`

### Unity Package (Recommended)

For full Unity integration (asset importers):

1. Open Package Manager: `Window > Package Manager`
2. Click `+ > Add package from git URL`
3. Enter: `https://github.com/nuskey8/Lua-CSharp.git?path=src/Lua.Unity/Assets/Lua.Unity`

This installs:
- `.lua` file importer (creates `LuaAsset`)
- `.luac` file importer (creates `LuacAsset`)
- Unity-specific implementations

## Lua Assets

After installation, `.lua` files can be imported as assets.

### Using LuaAsset

```csharp
using Lua;
using UnityEngine;

public class LuaRunner : MonoBehaviour
{
    public TextAsset luaScript;  // Drag .lua file here
    
    private LuaState _state;
    
    void Start()
    {
        _state = LuaState.Create();
        
        // Execute the script
        var results = _state.DoStringAsync(luaScript.text).Result;
    }
}
```

### LuaAsset Properties

```csharp
// Get the script text
string scriptText = luaScript.Text;

// Execute with cancellation
var results = await state.DoStringAsync(luaScript.Text, cancellationToken);
```

## Module Loading

### ResourcesModuleLoader

Load modules from Unity Resources:

```csharp
// Place scripts in Resources folder
// Resources/scripts/module.lua

state.ModuleLoader = new ResourcesModuleLoader();

-- In Lua
local module = require("scripts.module")
```

### AddressablesModuleLoader

Load modules from Addressables:

```csharp
// Requires Unity Addressables package
state.ModuleLoader = new AddressablesModuleLoader();

-- In Lua
local module = require("myaddressable")
```

### Custom Loader

Implement your own loader:

```csharp
public class CustomLoader : ILuaModuleLoader
{
    public bool Exists(string moduleName)
    {
        // Check if module exists
    }
    
    public async ValueTask<LuaModule> LoadAsync(string moduleName, CancellationToken ct)
    {
        // Load the module
        return new LuaModule(luaValue, moduleName);
    }
}
```

## Unity-Specific Platform

Use `UnityStandardIO` and `UnityApplicationOsEnvironment` for Unity integration:

```csharp
using Lua.Platforms;

var platform = new LuaPlatform(
    FileSystem: new FileSystem(),
    OsEnvironment: new UnityApplicationOsEnvironment(),
    StandardIO: new UnityStandardIO(),
    TimeProvider: TimeProvider.System);

var state = LuaState.Create(platform);
```

### UnityStandardIO

Redirects `print()` output to `Debug.Log()`:

```csharp
// In Lua
print("Hello from Lua!")  // Appears in Unity Console
```

### UnityApplicationOsEnvironment

- Environment variables via Dictionary
- `os.exit()` calls `Application.Quit()`

## Async in Unity

Use `UniTask` for Unity's async pattern (if available):

```csharp
// With UniTask
public async UniTask RunScript()
{
    var state = LuaState.Create();
    await state.DoStringAsync("print('Hello')");
}
```

Or use standard async with coroutines:

```csharp
IEnumerator RunLua()
{
    var state = LuaState.Create();
    var task = state.DoStringAsync("print('Hello')");
    
    while (!task.IsCompleted)
        yield return null;
    
    var results = task.Result;
}
```

## Coroutine Integration

Use Lua coroutines with Unity:

```csharp
// Define a wait function
state.Environment["wait"] = new LuaFunction(async (context, ct) =>
{
    var seconds = context.GetArgument<double>(0);
    await Task.Delay(TimeSpan.FromSeconds(seconds), ct);
    return new ValueTask<int>(0);
});
```

In Lua:
```lua
print("start")
wait(1)  -- non-blocking
print("done")
```

## Debugging

### Debug.Log Integration

`UnityStandardIO` outputs to console:

```csharp
// All print() calls appear in Console
await state.DoStringAsync("print('Debug info')");
```

### Error Handling

```csharp
try
{
    await state.DoStringAsync(luaCode);
}
catch (LuaRuntimeException ex)
{
    Debug.LogError($"Lua error: {ex.Message}");
}
catch (LuaCompileException ex)
{
    Debug.LogError($"Lua compile error: {ex.Message}");
}
```

## Best Practices

### Performance

- Reuse `LuaState` instances when possible
- Use `await` for I/O operations
- Dispose states when done

### Script Organization

```
Assets/
├── Scripts/
│   └── MyLuaRunner.cs
├── Resources/
│   └── lua/
│       ├── main.lua
│       └── modules/
│           └── utils.lua
└── StreamingAssets/
    └── data.lua
```

### Loading Patterns

```csharp
// From Resources
var asset = Resources.Load<TextAsset>("lua/main");
var results = await state.DoStringAsync(asset.text);

// From ScriptableObject
public class LuaScript : ScriptableObject
{
    public string code;
}

// Usage
var script = Resources.Load<LuaScript>("Scripts/myScript");
await state.DoStringAsync(script.code);
```

## IL2CPP Support

Lua-CSharp works with IL2CPP because it's fully managed C# without native interop. This means:

- No P/Invoke
- No native libraries
- Full AOT compatibility
- Works in built players

## Example: Complete Unity Setup

```csharp
using System.Collections;
using Lua;
using Lua.Platforms;
using UnityEngine;

public class LuaManager : MonoBehaviour
{
    private LuaState _state;
    
    void Awake()
    {
        // Create platform with Unity integration
        var platform = new LuaPlatform(
            FileSystem: new FileSystem(),
            OsEnvironment: new UnityApplicationOsEnvironment(),
            StandardIO: new UnityStandardIO(),
            TimeProvider: TimeProvider.System);
        
        _state = LuaState.Create(platform);
        _state.OpenStandardLibraries();
        
        // Add custom functions
        _state.Environment["log"] = new LuaFunction((context, ct) =>
        {
            var message = context.GetArgument<string>(0);
            Debug.Log($"[Lua] {message}");
            return new ValueTask<int>(0);
        });
        
        _state.Environment["wait"] = new LuaFunction(async (context, ct) =>
        {
            var seconds = context.GetArgument<double>(0);
            await Task.Delay(TimeSpan.FromSeconds(seconds), ct);
            return new ValueTask<int>(0);
        });
    }
    
    public async void RunScript(string code)
    {
        try
        {
            await _state.DoStringAsync(code);
        }
        catch (System.Exception ex)
        {
            Debug.LogError($"Lua Error: {ex.Message}");
        }
    }
    
    void OnDestroy()
    {
        _state?.Dispose();
    }
}
```

## Troubleshooting

### "Unable to find LuaAsset"

Make sure the `.lua` file is in a folder that Unity imports (like `Assets/` or `Resources/`).

### Module Not Found

Check that your module loader is set and the path is correct.

### Performance Issues

- Use `ResourcesModuleLoader` for simple projects
- Consider caching loaded modules
- Use async/await to prevent frame drops

## Next Steps

- [Source Generator](source-generator.md) - Auto-generate C# wrappers
- [Platform & Sandboxing](platform-sandboxing.md) - Security features
- [Exception Handling](exception-handling.md) - Error handling