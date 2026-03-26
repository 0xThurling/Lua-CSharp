# Standard Libraries

Lua-CSharp implements Lua 5.2 standard libraries. This document covers all available standard library functions.

## Enabling Standard Libraries

Open all standard libraries at once:

```csharp
var state = LuaState.Create();
state.OpenStandardLibraries();
```

Or open individual libraries:

```csharp
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

## Basic Library (base)

The basic library provides fundamental functions.

### assert

```lua
assert(v [, message])
```

Raises an error if `v` is false/nil. Otherwise returns all arguments.

```lua
assert(true)                    -- OK
assert(false, "failed")        -- Error: failed
```

### collectgarbage

```lua
collectgarbage([opt [, arg]])
```

Performs garbage collection. In Lua-CSharp, this calls .NET's `GC.Collect()`.

### dofile

```lua
dofile([filename])
```

Opens and executes the named file. Returns all values from the script.

```lua
-- myscript.lua
return 1, 2, 3

-- In Lua
local a, b, c = dofile("myscript.lua")  -- a=1, b=2, c=3
```

### error

```lua
error(message [, level])
```

Raises a runtime error with the given message.

### getmetatable

```lua
getmetatable(object)
```

Returns the metatable of the object. Returns `nil` if no metatable.

```lua
local t = {}
local mt = { __add = function(a, b) return a.x + b.x end }
setmetatable(t, mt)
print(getmetatable(t))  -- table: ...
```

### ipairs

```lua
ipairs(t)
```

Returns an iterator for iterating over array elements.

```lua
local t = {"a", "b", "c"}
for i, v in ipairs(t) do
    print(i, v)  -- 1 a, 2 b, 3 c
end
```

### load / loadfile

```lua
load(chunk [, chunkname [, mode [, env]]])
loadfile([filename [, mode [, env]]])
```

Loads code without executing it.

```lua
local f = load("return 1 + 2")
print(f())  -- 3
```

### next

```lua
next(table [, index])
```

Returns next key-value pair in table iteration.

### pairs

```lua
pairs(t)
```

Returns an iterator for key-value pairs.

```lua
local t = {a = 1, b = 2}
for k, v in pairs(t) do
    print(k, v)  -- a 1, b 2
end
```

### pcall

```lua
pcall(f, arg1, ...)
```

Protected call - executes function and catches errors.

```lua
local ok, err = pcall(function()
    error("oops")
end)
print(ok, err)  -- false oops
```

### print

```lua
print(...)
```

Prints values to stdout (via `LuaPlatform.StandardIO`).

### rawequal / rawget / rawlen / rawset

```lua
rawequal(v1, v2)
rawget(table, index)
rawlen(v)
rawset(table, index, value)
```

Direct table operations without metamethods.

### select

```lua
select(index, ...)
```

Returns elements after index, or count with "#".

```lua
print(select("#", 1, 2, 3))  -- 3
print(select(2, 1, 2, 3))    -- 2 3
```

### setmetatable

```lua
setmetatable(table, metatable)
```

Sets the metatable of a table.

### tonumber

```lua
tonumber(e [, base])
```

Converts to number. Supports base 2-36 and hex.

```lua
print(tonumber("42"))       -- 42
print(tonumber("FF", 16))  -- 255
print(tonumber("0xFF"))    -- 255
```

### tostring

```lua
tostring(v)
```

Converts value to string. Uses `__tostring` metamethod if defined.

### type

```lua
type(v)
```

Returns the type of the value: "nil", "boolean", "number", "string", "table", "function", "thread", "userdata".

### xpcall

```lua
xpcall(f, msgh)
```

Protected call with custom error handler.

```lua
local ok, result = xpcall(function()
    error("oops")
end, function(err)
    return "Error: " .. err
end)
print(ok, result)  -- false Error: oops
```

## Mathematics Library (math)

### Constants

```lua
math.pi   -- 3.141592653589793
math.huge -- positive infinity
```

### Functions

```lua
math.abs(x)           -- absolute value
math.acos(x)          -- arc cosine
math.asin(x)          -- arc sine
math.atan(x)          -- arc tangent
math.atan2(y, x)      -- arc tangent of y/x
math.ceil(x)          -- ceiling
math.cos(x)           -- cosine
math.cosh(x)          -- hyperbolic cosine
math.deg(x)           -- radians to degrees
math.exp(x)           -- e^x
math.floor(x)         -- floor
math.fmod(x, y)       -- remainder of x/y
math.frexp(x)         -- mantissa and exponent
math.ldexp(m, e)      -- m * 2^e
math.log(x [, base])  -- logarithm
math.max(x, ...)      -- maximum
math.min(x, ...)      -- minimum
math.modf(x)          -- integer and fractional parts
math.pow(x, y)        -- x^y
math.rad(x)           -- degrees to radians
math.random([m [, n]]) -- random number
math.randomseed(x)    -- set random seed
math.sin(x)           -- sine
math.sinh(x)          -- hyperbolic sine
math.sqrt(x)          -- square root
math.tan(x)           -- tangent
math.tanh(x)          -- hyperbolic tangent
```

### Examples

```lua
print(math.sqrt(16))        -- 4
print(math.random(1, 10))  -- random 1-10
math.randomseed(os.time())
print(math.sin(math.pi/2)) -- 1
```

## String Library (string)

### Functions

```lua
string.byte(s [, i [, j]])     -- char codes
string.char(...)              -- codes to chars
string.dump(f)               -- function to binary
string.find(s, p [, init [, plain]])  -- find pattern
string.format(f, ...)        -- formatted string
string.gmatch(s, p)          -- global iterator
string.gsub(s, p, r [, n])    -- global substitute
string.len(s)                 -- length (UTF-16)
string.lower(s)              -- lowercase
string.match(s, p [, init])  -- match once
string.pack(fmt, ...)        -- pack to binary
string.packsize(fmt)         -- pack size
string.rep(s, n)             -- repeat
string.reverse(s)           -- reverse
string.sub(s, i [, j])       -- substring
string.unpack(fmt, s [, pos]) -- unpack binary
string.upper(s)              -- uppercase
```

### Pattern Matching

Lua string patterns are powerful:

```lua
-- Find and extract
local s = "hello world"
local i, j = string.find(s, "world")
print(i, j)  -- 7 11

-- Captures
local date = "Today: 2024-01-15"
local year, month, day = string.match(date, "(%d+)-(%d+)-(%d+)")
print(year, month, day)  -- 2024 01 15

-- Global substitution
local s = string.gsub("hello world", "world", "lua")
print(s)  -- hello lua

-- Patterns
local words = {}
for w in string.gmatch("one two three", "%w+") do
    table.insert(words, w)
end
```

### Format Specifiers

```lua
-- %d - integer
-- %s - string
-- %f - float
-- %x - hex
-- %o - octal
-- %.2f - 2 decimal places

string.format("%s: %.2f", "Price", 9.99)  -- Price: 9.99
```

## Table Library (table)

### Functions

```lua
table.concat(list [, sep [, i [, j]]])  -- concatenate
table.insert(list, [pos,] value)        -- insert
table.move(a1, a2, f, t [, table])     -- move
table.pack(...)                        -- pack to table
table.remove(list [, pos])             -- remove
table.sort(list [, comp])              -- sort
table.unpack(list [, i [, j]])         -- unpack
```

### Examples

```lua
-- Concatenate
local t = {"a", "b", "c"}
print(table.concat(t, ", "))  -- a, b, c

-- Insert and remove
table.insert(t, "d")
table.remove(t, 2)

-- Sort with custom comparator
table.sort(t, function(a, b) return a > b end)

-- Pack/Unpack
local args = table.pack(1, 2, 3)
print(args[1], args.n)  -- 1 3
local a, b, c = table.unpack(args)
```

## Coroutine Library (coroutine)

### Functions

```lua
coroutine.create(f)           -- create
coroutine.isyieldable(co)    -- check if can yield
coroutine.resume(co, ...)    -- start/continue
coroutine.running()          -- current
coroutine.status(co)         -- status
coroutine.wrap(f)            -- wrapped function
coroutine.yield(...)         -- suspend
```

### Examples

```lua
-- Create and resume
local co = coroutine.create(function()
    print("running")
    coroutine.yield(1)
    print("done")
end)

print(coroutine.resume(co))  -- true 1
print(coroutine.resume(co))  -- true (no value)
print(coroutine.status(co))  -- dead
```

### Producer-Consumer

```lua
local function producer()
    for i = 1, 5 do
        coroutine.yield(i)
    end
end

local p = coroutine.create(producer)
while coroutine.resume(p) do end

-- Or using wrap
local p = coroutine.wrap(producer)
while true do
    local v = p()
    if not v then break end
    print(v)
end
```

## I/O Library (io)

### Functions

```lua
io.close([file])          -- close
io.flush()               -- flush output
io.input([file])         -- set input
io.lines([filename])      -- iterate lines
io.open(filename, mode)  -- open file
io.output([file])         -- set output
io.popen(cmd, mode)      -- pipe (if supported)
io.read(...)              -- read
io.tmpfile()             -- temp file
io.type(obj)             -- file type
io.write(...)            -- write
```

### File Modes

- `r` - read
- `w` - write (truncate)
- `a` - append
- `r+` - read/write
- `w+` - read/write (truncate)
- `a+` - append/read/write

### Examples

```lua
-- Read file
local f = io.open("test.txt", "r")
local content = f:read("*all")
f:close()

-- Write file
local f = io.open("out.txt", "w")
f:write("Hello\n")
f:close()

-- Lines
for line in io.lines("file.txt") do
    print(line)
end
```

## Operating System Library (os)

### Functions

```lua
os.clock()           -- CPU time
os.date([format [, time]])  -- date/time
os.difftime(t1, t2)  -- time difference
os.execute(cmd)      -- shell command
os.exit([code [, close]]) -- exit
os.getenv(name)      -- env variable
os.remove(filename) -- delete file
os.rename(old, new) -- rename
os.setlocale(loc [, cat]) -- locale
os.time([table])    -- timestamp
os.tmpname()        -- temp name
```

### Examples

```lua
print(os.date())                    -- current date
print(os.date("%Y-%m-%d"))         -- formatted
print(os.time())                    -- Unix timestamp

local t = os.date("*t")
print(t.year, t.month, t.day)
```

## Module Library (package)

### Variables

```lua
package.loaded   -- loaded modules
package.preload  -- preload table
package.path     -- module search path
package.searchers  -- search functions
package.searchpath(name, path, sep, rep)  -- search path
```

### require

```lua
require(modname)
```

Loads a module.

```lua
-- Search order:
-- 1. package.preload[modname]
-- 2. package.searchers (custom loaders)
-- 3. package.path files (?.lua)
```

### Searchers

Custom searchers can be added to `package.searchers`.

## Bit32 Library (bit32)

### Functions

```lua
bit32.arshift(x, n)    -- arithmetic right shift
bit32.band(...)        -- bitwise AND
bit32.bnot(x)          -- bitwise NOT
bit32.bor(...)        -- bitwise OR
bit32.btest(...)       -- test AND
bit32.bxor(...)        -- bitwise XOR
bit32.extract(n, f, w) -- extract bits
bit32.lrotate(x, n)    -- left rotate
bit32.lshift(x, n)     -- left shift
bit32.replace(n, v, f, w) -- replace bits
bit32.rrotate(x, n)   -- right rotate
bit32.rshift(x, n)     -- right shift
```

### Examples

```lua
print(bit32.band(0xFF, 0x0F))  -- 15
print(bit32.bor(0x0F, 0xF0))  -- 255
print(bit32.lshift(1, 4))     -- 16
```

## Debug Library (debug)

> [!CAUTION]
> The debug library is primarily for debugging and may impact performance.

### Functions

```lua
debug.debug()              -- interactive debug
debug.gethook()            -- get hook
debug.getinfo(f [, what])  -- function info
debug.getlocal(f, n)       -- local variable
debug.getmetatable(obj)    -- get metatable
debug.getupvalue(f, n)     -- upvalue
debug.getuservalue(u)      -- userdata value
debug.sethook(hook, mask [, count]) -- set hook
debug.setlocal(f, n, v)    -- set local
debug.setmetatable(obj, mt) -- set metatable
debug.setupvalue(f, n, v) -- set upvalue
debug.setuservalue(u, v)   -- set userdata value
debug.traceback([msg [, level]]) -- traceback
debug.upvalueid(f, n)     -- upvalue id
debug.upvaluejoin(f1, n1, f2, n2) -- join upvalues
```

### Examples

```lua
-- Traceback
local ok, err = pcall(function()
    error("oops")
end)
print(debug.traceback(err))

-- Function info
local function foo() end
local info = debug.getinfo(foo)
print(info.what, info.name, info.source)
```

## Compatibility Notes

### UTF-16 Strings

Lua-CSharp uses UTF-16 encoding for strings, which differs from standard Lua (UTF-8). This affects `string.len()`:

```lua
-- Standard Lua: 5
-- Lua-CSharp: 15 (character count in UTF-16)
local l = string.len("あいうえお")
```

### Garbage Collection

`collectgarbage()` calls .NET's garbage collector, which may behave differently than Lua's GC.

### File I/O

File operations use `LuaPlatform.FileSystem`, which can be customized for sandboxing.