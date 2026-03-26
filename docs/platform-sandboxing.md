# Platform & Sandboxing

Lua-CSharp provides a powerful platform abstraction system for sandboxing and customizing the Lua runtime environment.

## LuaPlatform Overview

`LuaPlatform` encapsulates all external dependencies that Lua uses:

- File system access
- OS environment (environment variables)
- Standard input/output
- Time provider

```csharp
using Lua.Platforms;

var platform = new LuaPlatform(
    FileSystem: fileSystem,
    OsEnvironment: osEnvironment,
    StandardIO: standardIO,
    TimeProvider: TimeProvider.System);

var state = LuaState.Create(platform);
```

## Components

### ILuaFileSystem

Controls file access for `require`, `dofile`, and I/O operations:

```csharp
public interface ILuaFileSystem
{
    bool FileExists(string path);
    bool DirectoryExists(string path);
    ILuaStream? OpenRead(string path);
    ILuaStream? OpenWrite(string path);
    ILuaStream? OpenReadWrite(string path);
    void Delete(string path);
    void Move(string src, string dst);
    // ... more methods
}
```

### ILuaOsEnvironment

Controls OS-level operations:

```csharp
public interface ILuaOsEnvironment
{
    string? GetEnvironmentVariable(string name);
    void SetEnvironmentVariable(string name, string? value);
    IEnumerable<string> GetEnvironmentVariables();
    int System(string command);
    void Exit(int code);
    string? GetWorkingDirectory();
    long TickCount { get; }
}
```

### ILuaStandardIO

Controls input/output:

```csharp
public interface ILuaStandardIO
{
    ILuaStream Input { get; }
    ILuaStream Output { get; }
    ILuaStream Error { get; }
}
```

### TimeProvider

Controls time-related operations (from `System.TimeProvider`).

## Creating a Sandbox

### Restricted File System

```csharp
public class RestrictedFileSystem : ILuaFileSystem
{
    readonly string _rootDirectory;
    
    public RestrictedFileSystem(string rootDirectory)
    {
        _rootDirectory = Path.GetFullPath(rootDirectory);
    }
    
    private string ResolvePath(string path)
    {
        var fullPath = Path.GetFullPath(Path.Combine(_rootDirectory, path));
        if (!fullPath.StartsWith(_rootDirectory))
            throw new UnauthorizedAccessException("Path escape attempt");
        return fullPath;
    }
    
    public bool FileExists(string path)
    {
        return File.Exists(ResolvePath(path));
    }
    
    public bool DirectoryExists(string path)
    {
        return Directory.Exists(ResolvePath(path));
    }
    
    // Implement other methods...
}
```

### Restricted OS Environment

```csharp
public class RestrictedOsEnvironment : ILuaOsEnvironment
{
    readonly Dictionary<string, string> _variables = new();
    
    public string? GetEnvironmentVariable(string name)
    {
        return _variables.TryGetValue(name, out var value) ? value : null;
    }
    
    public void SetEnvironmentVariable(string name, string? value)
    {
        if (value == null)
            _variables.Remove(name);
        else
            _variables[name] = value;
    }
    
    public IEnumerable<string> GetEnvironmentVariables() => _variables.Keys;
    
    public int System(string command) => throw new UnauthorizedAccessException();
    
    public void Exit(int code) => throw new UnauthorizedAccessException();
    
    public string? GetWorkingDirectory() => "/";
    
    public long TickCount => Environment.TickCount;
}
```

### Restricted Standard I/O

```csharp
public class RestrictedStandardIO : ILuaStandardIO
{
    readonly StringBuilder _output = new();
    
    public ILuaStream Input => throw new UnauthorizedAccessException();
    
    public ILuaStream Output => new StringWriter(_output);
    
    public ILuaStream Error => new StringWriter(_output);
}
```

### Complete Sandbox

```csharp
public LuaPlatform CreateSandbox()
{
    return new LuaPlatform(
        FileSystem: new RestrictedFileSystem("/sandbox"),
        OsEnvironment: new RestrictedOsEnvironment(),
        StandardIO: new RestrictedStandardIO(),
        TimeProvider: TimeProvider.System);
}

// Usage
var sandbox = CreateSandbox();
var state = LuaState.Create(sandbox);
```

## Module Loading Control

### Custom Module Loader

Restrict what modules can be loaded:

```csharp
public class AllowedModulesLoader : ILuaModuleLoader
{
    readonly HashSet<string> _allowed = new(StringComparer.Ordinal)
    {
        "json",
        "math",
        "string"
    };
    
    public bool Exists(string moduleName)
    {
        return _allowed.Contains(moduleName);
    }
    
    public async ValueTask<LuaModule> LoadAsync(string moduleName, CancellationToken ct)
    {
        if (!_allowed.Contains(moduleName))
            throw new LuaModuleNotFoundException(moduleName);
        
        // Load the module...
    }
}

// Usage
var state = LuaState.Create(platform);
state.ModuleLoader = new AllowedModulesLoader();
```

### Composite Module Loader

Combine multiple loaders with priority:

```csharp
state.ModuleLoader = CompositeModuleLoader.Create(
    new CustomModuleLoader(),    // Custom modules first
    new ResourcesModuleLoader()  // Then Unity Resources
);
```

## Time Control

### Simulated Time

```csharp
public class FakeTimeProvider : TimeProvider
{
    long _ticks;
    
    public override long GetTimestamp() => _ticks;
    
    public override TimeSpan GetElapsedTime(TimeSpan startTime)
    {
        return TimeSpan.FromTicks(_ticks - startTime.Ticks);
    }
    
    public void Advance(TimeSpan duration)
    {
        _ticks += duration.Ticks;
    }
}
```

## Security Considerations

### Path Traversal

Always validate file paths:

```csharp
private string SafePath(string path)
{
    var fullPath = Path.GetFullPath(Path.Combine(_rootDir, path));
    if (!fullPath.StartsWith(_rootDir))
        throw new UnauthorizedAccessException();
    return fullPath;
}
```

### Command Injection

Block `os.execute` and similar:

```csharp
public int System(string command)
{
    // Block all system commands
    throw new UnauthorizedAccessException("System commands disabled");
}
```

### Resource Limits

```csharp
public class LimitedFileSystem : ILuaFileSystem
{
    int _readCount;
    const int MaxReads = 100;
    
    public ILuaStream? OpenRead(string path)
    {
        if (_readCount++ > MaxReads)
            throw new IOException("Too many file operations");
        // ...
    }
}
```

## Default Platform

For development/testing:

```csharp
// Default: full access
var state = LuaState.Create();  // Uses LuaPlatform.Default
```

## Platform for Different Environments

### Console Application

```csharp
var platform = new LuaPlatform(
    FileSystem: new FileSystem(),
    OsEnvironment: new SystemOsEnvironment(),
    StandardIO: new ConsoleStandardIO(),
    TimeProvider: TimeProvider.System);
```

### Web Server (Restricted)

```csharp
var platform = new LuaPlatform(
    FileSystem: new ReadOnlyFileSystem(webRoot),
    OsEnvironment: new RestrictedOsEnvironment(),
    StandardIO: new HttpStandardIO(),
    TimeProvider: TimeProvider.System);
```

### Unity

```csharp
var platform = new LuaPlatform(
    FileSystem: new FileSystem(),
    OsEnvironment: new UnityApplicationOsEnvironment(),
    StandardIO: new UnityStandardIO(),
    TimeProvider: TimeProvider.System);
```

## Best Practices

1. **Always create a new platform** for each sandboxed environment
2. **Validate all paths** to prevent traversal attacks
3. **Limit file operations** to prevent DoS
4. **Block dangerous OS operations** in production
5. **Use readonly where possible** to prevent state changes

## Example: Complete Web Sandbox

```csharp
public class WebLuaSandbox
{
    public static LuaState Create(string webRoot)
    {
        var platform = new LuaPlatform(
            FileSystem: new SecureFileSystem(webRoot),
            OsEnvironment: new ReadOnlyOsEnvironment(),
            StandardIO: new LogOnlyStandardIO(),
            TimeProvider: TimeProvider.System);
        
        var state = LuaState.Create(platform);
        
        // Add allowed modules
        state.ModuleLoader = CompositeModuleLoader.Create(
            new SafeModuleLoader(),
            new FileBasedModuleLoader(webRoot));
        
        return state;
    }
}

public class SecureFileSystem : ILuaFileSystem
{
    readonly string _root;
    
    public SecureFileSystem(string root)
    {
        _root = Path.GetFullPath(root);
    }
    
    // Implementation with path validation...
}

// Usage
using var state = WebLuaSandbox.Create("/var/www/scripts");
var results = await state.DoFileAsync("user_script.lua");
```

## Next Steps

- [Exception Handling](exception-handling.md) - Error handling patterns
- [Unity Integration](unity-integration.md) - Unity-specific platform
- [API Reference](api-reference.md) - Complete API