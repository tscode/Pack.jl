
# Pack.jl

This package is for people who want to serialize their beloved structures in
a simple and transparent way. It operates on top of the
[msgpack format](https://msgpack.org/index.html).

You might like this package because it

 - is pure lightweight julia with no dependencies.
 - does not allow uncontrolled code execution during unpacking.
 - is very flexible, with context-dependent format choices.
 - is quite fast for most objects.

On the other hand, Pack.jl is probably not the right choice if you
 
 - need to serialize arbitrary julia objects out-of-the-box.
 - have enormous files and need lazy loading capabilities.
 - want to read arbitrary msgpack files from external sources.

While the functionality to read generic msgpack is included to some degree
(currently without support for extensions), the excellent package
[MsgPack.jl]() is probably a better choice for this task.

## Usage

The functionality of `Pack.jl` revolves around the three functions `pack`,
`unpack`, and `format`. The latter decides how a julia value should be converted
to a binary representation during calls of the former two.

Formats are realized as singleton subtypes of `Pack.Format`. Core formats, that
correspond more or less directly to underlying msgpack formats, are `NilFormat`,
`BoolFormat`, `SignedFormat`, `UnsignedFormat`, `FloatFormat`, `VectorFormat`,
`MapFormat`, and `BinaryFormat`.

Convenience formats built on top of these are `ArrayFormat`, `BinVectorFormat`,
`BinArrayFormat`, as well as `TypedFormat`. The corresponding docstrings provide
additional information.

Many base types in julia have a default format associated to them and can be
packed / unpacked without further instructions.
```julia
import Pack

str = "This is a string" # Pack.StringFormat
tup = ("tuple", 5, false) # Pack.VectorFormat
ntup = (a = str, b = tup) # Pack.MapFormat

bytes = Pack.pack(ntup)
Pack.unpack(bytes, typeof(ntup))

# or alternatively 

io = IOBuffer()
Pack.pack(io, ntup)
Pack.unpack(io, typeof(ntup))
```
The type information passed to `Pack.unpack` is crucial, since (by default) no
type information is stored in `bytes`. If it is left out, `Pack.unpack(bytes)`
tries to load `bytes` as generic msgpack object and returns a `Dict`.

See the section ?? below on how to unpack objects `::T` when `T` is an abstract type.

### Custom packing

If you have defined your own struct and want to serialize each field via the
default formats, you have two immediate options to do so.
```julia
struct MyStruct
  a::Float64
  b::String
end

Pack.format(::MyStruct) = Pack.MapFormat     # save the field names a and b
# or
Pack.format(::MyStruct) = Pack.VectorFormat  # do not save the field names
```
Pack.jl also provides the macro `@pack` that can conveniently be used to
declare default formats: `Pack.@pack MyStruct in Pack.MapFormat`.

This macro has additional benefits: If you do not want to store all fields of
MyStruct, or use a specific constructor when unpacking, you can easily inform
`@pack` in combination with `MapFormat`.
```julia
# Special constructor we want to use during unpacking
Mystruct(a; b) = MyStruct(a, b)
Pack.@pack Mystruct in MapFormat MyStruct(a; b)

# or

# We only want to serialize the field a
Mystruct(a) = MyStruct(a, "always the same")
Pack.@pack MyStruct in MapFormat MyStruct(a)
```
With a bit of additional work we have even more flexibility. For example, after
we have decided to store only the field `MyStruct.a::Float64` anyway, we can
just use `FloatFormat`.
```julia
Pack.format(::Type{MyStruct}) = FloatFormat()
Pack.deconstruct(val::MyStruct, ::FloatFormat) = val.a
Pack.construct(val::MyStruct, a, ::FloatFormat) = MyStruct(a, "always the same")
```
See the docstrings for the various supported formats to learn about the
respective requirements for `destruct` and `construct`.

### Context matters: Fields

Pack.jl strives to be flexible when handling custom structs. In particular, it
disagrees that a given value `v::T` should always be serialized in the same way,
independent of the context.

There are two primary mechanism to attach context information to `v::T` while it
is being packed or unpacked: via its (potential) *parent* or via *scopes*.

Here is a simple example where a field (the child) of a struct (the parent)
receives a non-default format.

```julia
import Pack: MapFormat, BinVectorFormat

struct MyStruct
  a::String           # Default format is Pack.StringFormat
  b::Vector{Float32}  # Default format is Pack.VectorFormat
end

# We decide that MyStruct.b should rather be stored as binary vector
Pack.@pack MyStruct in MapFormat (b in BinVectorFormat)

value = MyStruct("My data is stored in binary", rand(Float32, 10))
bytes = pack(value)     # serialize the structure
unpack(bytes, MyStruct) # unpack the object again
```
Under the hood, this behavior is implemented by specializing the function
`valueformat`, which determines the formats to be used for struct fields.
For example, the following code reproduces the call to `@pack` shown above:
```julia
Pack.format(::Type{MyStruct}) = MapFormat()

function Pack.valueformat(::Type{MyStruct}, index)
  index == 2 ? BinVectorFormat() : DefaultFormat()
end
```

### Context matters: Scopes

Another supported way to modify the format of `v::T` in a call to `pack` or
`unpack` are so-called *scopes*, realized as singleton subtypes of `Scope`.
Scopes are particularly useful if you desired to change how `v::T` is serialized
but are reluctant to modify the global behavior for all values of type `T` (for
example, because `::T` belongs to a third party package and you do not want to
mess with its defaults).

Scopes are most conveniently created with the macro `Pack.@scope`.
```julia
import Pack: ArrayFormat

# In this scope, MyStruct.b is packed via ArrayFormat
scope = Pack.@scope MyStruct in MapFormat (b in ArrayFormat)

value = MyStruct("My data is stored as array", rand(Float32, 10))
bytes = pack(value, scope)
unpack(bytes, MyStruct, scope)
```
Scopes can be used to influence each aspect of the serialization, since they
penetrate each packing related call (`pack`, `unpack`, `format`, `valueformat`,
and so on). In order to add a rule to a scope, just specialize the respective
method.

Without using macros, the example above can be reproduced as follows:
```julia
struct MyScope <: Scope end

format(::Type{MyStruct}, ::MyScope) = MapFormat()

function Pack.valueformat(::Type{MyStruct}, index, ::MapFormat, ::MyScope)
  index == 2 ? ArrayFormat() : DefaultFormat()
end

bytes = pack(value, MyScope())
unpack(bytes, MyStruct, MyScope())
```

### Unpacking the abstract


