# Lua Lib Class

A small Lua OOP helper that adds:

- Named classes
- Single inheritance
- Java-style interfaces
- `super()` calls
- `instanceOf()` checks
- `implements()` checks
- Callable classes
- Lua metamethod forwarding for operator overloading
- used in [`raymp`](https://github.com/abdorayden/raymp)

The module exports a single table named `OOP` with two constructors:

- `OOP.class(name, superClass?, ...interfaces)`
- `OOP.interface(name, ...methods)`

## Installation

Copy [`oop.lua`](./oop.lua) into your project and require it:

```lua
local OOP = require("oop")
```

## Quick Start

```lua
local OOP = require("oop")

local Animal = OOP.class("Animal")

function Animal:constructor(name)
    self.name = name
end

function Animal:speak()
    return "..."
end

local Dog = OOP.class("Dog", Animal)

function Dog:constructor(name, breed)
    self:super("constructor", name)
    self.breed = breed
end

function Dog:speak()
    return "Woof!"
end

local dog = Dog("Buddy", "Golden Retriever")

print(dog.name)                  -- Buddy
print(dog:speak())               -- Woof!
print(dog:instanceOf(Dog))       -- true
print(dog:instanceOf(Animal))    -- true
```

## API

### `OOP.interface(name, ...methods)`

Creates an interface descriptor.

```lua
local Drawable = OOP.interface("Drawable", "draw", "resize")
```

Arguments:

- `name`: interface name used in error messages
- `...methods`: method names that implementing classes must define

Returned interface fields:

- `name`
- `methods`
- `__type == "interface"`

Interfaces are validated when an instance is created with `.new(...)` or `Class(...)`.

If a class does not implement every required method, construction fails with an error similar to:

```text
Class 'Circle' must implement interface 'Drawable'. Missing methods: draw, resize
```

### `OOP.class(name, superClass?, ...interfaces)`

Creates a class.

```lua
local Shape = OOP.class("Shape")
local Circle = OOP.class("Circle", Shape, Drawable)
```

Arguments:

- `name`: class name
- `superClass`: optional parent class
- `...interfaces`: zero or more interfaces created with `OOP.interface`

Returned class fields:

- `__name`
- `__super`
- `__interfaces`
- `__type == "class"`
- `__index`
- `new(...)`

## Creating Instances

Two instance creation styles are supported:

```lua
local obj1 = MyClass.new(...)
local obj2 = MyClass(...)
```

Both forms:

- validate interface methods
- create a new table instance
- call `constructor(...)` if present
- attach supported metamethods from the class onto the instance metatable

## Constructors

If a class defines `constructor`, it is called automatically when an instance is created.

```lua
local User = OOP.class("User")

function User:constructor(name)
    self.name = name
end
```

There is no special constructor return value. Initialize fields on `self`.

## Inheritance

Inheritance is single-parent only.

```lua
local Vehicle = OOP.class("Vehicle")
local Car = OOP.class("Car", Vehicle)
```

Method lookup flows through the parent chain using metatables, so child classes inherit parent methods automatically unless they override them.

## Calling Parent Methods with `super`

Use `self:super(methodName, ...)` inside instance methods.

```lua
function Car:constructor(make, model)
    self:super("constructor", make)
    self.model = model
end

function Car:start()
    self:super("start")
    print("Car ready")
end
```

Behavior:

- It finds the class that called `super`
- It walks upward from that class's parent
- It calls the first superclass method with the requested name

This makes overridden methods and multi-level inheritance work as expected.

If no parent implementation exists, it raises an error.

## Type Checks

### `instance:instanceOf(Class)`

Returns `true` if the instance belongs to `Class` or any of its ancestors.

```lua
print(dog:instanceOf(Dog))     -- true
print(dog:instanceOf(Animal))  -- true
```

### `Class:implements(Interface)` or `instance:implements(Interface)`

Checks whether the class declares the interface directly or inherits it from a parent class.

```lua
local Drawable = OOP.interface("Drawable", "draw")
local Shape = OOP.class("Shape")
local Circle = OOP.class("Circle", Shape, Drawable)

function Circle:draw()
end

print(Circle:implements(Drawable))  -- true

local circle = Circle()
print(circle:implements(Drawable))  -- true
```

## Interfaces Example

```lua
local Drawable = OOP.interface("Drawable", "draw", "getDimensions")
local Resizable = OOP.interface("Resizable", "resize", "getScale")

local Shape = OOP.class("Shape")

function Shape:constructor(x, y)
    self.x = x or 0
    self.y = y or 0
end

local Circle = OOP.class("Circle", Shape, Drawable, Resizable)

function Circle:constructor(x, y, radius)
    self:super("constructor", x, y)
    self.radius = radius or 1
    self.scale = 1
end

function Circle:draw()
    print("drawing")
end

function Circle:getDimensions()
    return {
        radius = self.radius * self.scale,
        x = self.x,
        y = self.y
    }
end

function Circle:resize(factor)
    self.scale = factor
end

function Circle:getScale()
    return self.scale
end

local circle = Circle(10, 20, 5)
print(circle:implements(Drawable))   -- true
print(circle:implements(Resizable))  -- true
```

## Operator Overloading

The module copies common Lua metamethods from the class table to each created instance, so operator overloading works when those methods are defined on the class.

Supported metamethods include:

- `__add`
- `__sub`
- `__mul`
- `__div`
- `__mod`
- `__pow`
- `__unm`
- `__concat`
- `__len`
- `__eq`
- `__lt`
- `__le`
- `__tostring`
- `__pairs`
- `__ipairs`
- `__gc`
- `__mode`
- `__metatable`
- `__idiv`
- `__band`
- `__bor`
- `__bxor`
- `__bnot`
- `__shl`
- `__shr`

Example:

```lua
local Vector = OOP.class("Vector")

function Vector:constructor(x, y)
    self.x = x or 0
    self.y = y or 0
end

function Vector:__add(other)
    return Vector(self.x + other.x, self.y + other.y)
end

function Vector:__tostring()
    return string.format("Vector(%d, %d)", self.x, self.y)
end

local a = Vector(1, 2)
local b = Vector(3, 4)
local c = a + b

print(c) -- Vector(4, 6)
```

## Notes and Behavior Details

- Interface validation happens at instantiation time, not when the class is declared.
- A class can implement multiple interfaces.
- Inheritance is single-parent only.
- `super()` only works for methods that exist somewhere above the calling class.
- The implementation uses `debug.getinfo` to resolve the calling class for `super()`, with a fallback search for the named method in the inheritance chain.
- Classes are callable because the class metatable defines `__call` and forwards to `.new(...)`.

## Minimal Example

```lua
local OOP = require("oop")

local Named = OOP.interface("Named", "getName")

local Person = OOP.class("Person", nil, Named)

function Person:constructor(name)
    self.name = name
end

function Person:getName()
    return self.name
end

local p = Person("Alice")

print(p:getName())             -- Alice
print(p:instanceOf(Person))    -- true
print(p:implements(Named))     -- true
```

## License

MIT. See the license header in [`oop.lua`](./oop.lua).
