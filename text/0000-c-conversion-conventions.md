# Summary

This RFC summarizes the approach to binding C++ to C as currently taken by [cppmm](https://github.com/vfx-rs/cppmm). In particular it describes the rules for memory representation of C versions of C++ types, how they are named, and how they are interacted with by the binding layer.

cppmm works by traversing the AST of the binding file, first to make a tree of classes (and their methods) and functions to be bound by examining only the nodes in the cppmm_bind namespace, and then traversing the entire tree to find matching declarations in the bound library.

For each class, enum, method and function to be bound it generates a C-compatible name and wrapper type/function using a simple set of rules. This RFC seeks comment on those rules.

# Motivation

C++ allows for a variety of names that can't be mapped directly to C, e.g. con/destructors, operators and overloads. We should come up with a default naming convention that produces sensible names in these cases and can be enforced by cppmm in order to make all our C APIs consistent.

Since cppmm allows renaming functions using attributes it is trivial to change these in the binding file, but the rules decided upon here should be good enough for most cases by default and provide guidelines for binding authors in the cases where names can't be decided upon automatically.


# Memory layout of types
There are three different ways (**Kinds**) to represent a C++ type in C, which I'm currently calling:

> Note: in all the following code the `to_cpp()` and `to_c()` functions are automatically created in an anonymous namespace in the bindings in order to avoid lots of `reinterpret_cast<>()s` drowning out the actual code.

## 1. `opaqueptr`
A pointer to an undefined type. The C API has no idea about the memory representation of the type and just passes pointers around:
```c
typedef struct OIIO_ImageInput OIIO_ImageInput;
```
This has the knock-on effect that all instances of this type must be heap allocated. For large types this probably isn't an issue (ImageInput is always heap-allocated anyway) but for smaller types might introduce an unwelcome overhead.

## 2. `valuetype`
A straightforward copy of the type from C++ to C:
```c
typedef struct {
    unsigned char basetype;
    unsigned char aggregate;
    unsigned char vecsemantics;
    unsigned char reserved;
    int arraylen;
} OIIO_TypeDesc;
```
This obviously means that all fields must be valid C types. These can be created on the stack or the heap at the caller's discretion, and passed to C++ by value or by pointer.

## 3. `opaquebytes`
A type matching the memory layout of the C++ type it's representing, but with its fields hidden behind an opaque byte array:
```c
typedef struct {
    char _private[2];
} Imath_half
CPPMM_ALIGN(2);
```
These can be created on the stack or the heap by the caller as well. This is handy for types that have private fields that you don't want callers poking around in. It is also potentially possible to pass C++ types around in C directly like this, but there are some real safety issues with doing so related to RAII.

# Constructing/Destructing Objects
The rules for constructing new instances of types depend on their kind.

## 1. opaqueptr
This kind is constructed simply by calling new and delete.
```c++
// C++
namespace FOO {
struct Bar {
    Bar(int a);
    ~Bar();
} CPPMM_OPAQUEPTR;
}

// C
FOO_Bar* FOO_Bar_new(int a) {
    return to_c(new FOO::Bar(a));
}

void FOO_Bar_delete(FOO_Bar* self) {
    delete to_cpp(self);
}
```

## 2. opaquebytes and valutype
Since the caller allocates the memory, these are constructed using placement new, with the caller passing in a pointer to the pre-allocated region. Destructing means calling the destructor explicitly and the caller is responsible for freeing the associated memory.
```c++
// C++
namespace FOO {
struct Bar {
    Bar(int a);
    ~Bar();
} CPPMM_OPAQUEBYTES;
}

// C
FOO_Bar* FOO_Bar_ctor(FOO_Bar* self, int a) {
    return to_c( new (self) FOO::Bar(a));
}

void FOO_Bar_dtor(FOO_Bar* self) {
    to_cpp(self)->~Bar();
}
```

# Naming Rules

## 1. prepend entire namesace hierarchy to declaration name
Consider the following C++ code
```c++
namespace FOO {
struct Bar {
    enum BazThings { Thing1, Thing2 };
    void baz();
};

void qux();
}
```

This would generate the following C declarations:
```c
typedef struct FOO_Bar FOO_Bar;

enum FOO_Bar_BazThings { FOO_Bar_BazThings_Thing1, FOO_Bar_BazThings_Thing2 };

void FOO_Bar_baz(FOO_Bar* self);

void FOO_qux();
```

### Considerations
Most (all?) of the libraries we're interested in version their namespaces to avoid dependency hell. cppmm currently allows renaming namespaces with the `-n` flag to make nicer names, for instance:
```c
OIIO_ImageInput_open()
```
instead of:
```c
OpenImageIO_v2_8_ImageInput_open()
```
though this could also be shortened to:
```c
OIIO28_ImageInput_open()
```
or somesuch.

Unfortunately I think we might have to bake the version into the C API "namespaces" for exactly the same reason as in C++. Consider a Rust crate that transitively depends on two different versions of OIIO - might we get symbol clashing here? I'm not sure... pretty sure you would if the binding stubs were compiled as shared libraries, maybe not if they're static?

## 2. The "self" pointer in methods should be called "self"
Obviously `this` is taken and Rust and Python both use `self` so it should be familiar to everyone reading.

## 3. Constructors/destructors
### opaqueptr
Just use `new` and `delete`
```c
FOO_Bar* FOO_Bar_new();
FOO_Bar_delete(const FOO_Bar* self);
```

### opaquebytes and valuetype
Use `ctor` and `dtor` - these are short, unlikely to clash, and describe exactly what's going on:
```c
FOO_Bar* FOO_Bar_ctor(FOO_Bar* self);
FOO_Bar_dtor(FOO_Bar* self);
```

#### Alternatives:
<dl>
  <dt><tt>constructor</tt> / <tt>destructor</tt></dt>
  <dd>these are also good but a bit long</dd>
  <dt><tt>construct</tt> / <tt>destruct</tt></dt>
  <dd>OK but could be better used for value types?</dd>
  <dt><tt>create / destroy</tt></dt>
  <dd>These are extremely common in C++ interfaces so just asking for collisions</dd>
</dl>



### Constructor overloads

Use `from_<type>` or `with_<params>` depending on which makes more conceptual sense. If the constructor is essentially acting as a conversion and the type of the parameter is more important, choose `from`, if it's acting as a configuration and/or the name of the parameter is more important, choose `with`:

```c++
// C++
namespace Imath
class half {
    half(float f);
} CPPMM_OPAQUEBYTES;
}

// C
Imath_half_from_float(Imath_half* self, float f);
```

```c++
// C++
namespace OIIO
class ImageSpec {
    ImageSpec(int width, int height);
} CPPMM_OPAQUEPTR;
}

// C
OIIO_ImageSpec* OIIO_ImageSpec_with_dimensions(int width, int height);
```

### Copy constructor/assignment operator
These should just be `copy` and `assign`, respectively:
```c++
// C++
namespace Imath
class half {
    half(const half& other);
    half& operator=(const half& other);
} CPPMM_OPAQUEBYTES;
}

// C
Imath_half Imath_half_copy(Imath_half* self, const Imath_half* other);
Imath_half* Imath_half_assign(Imath_half* self, const Imath_half* other);
```

Move constructors and assignment operators shiould be ignored.


## 4. Operators
Rust's operator trait names are concise and memorable so let's just use those:
```c
// +
Imath_half Imath_half_add(const Imath_half* self, Imath_half other);
// +=
Imath_half* Imath_half_add_assign(Imath_half* self, Imath_half other);

// *
Imath_half Imath_half_mul(const Imath_half* self, Imath_half other);
// *=
Imath_half* Imath_half_mul_assign(Imath_half* self, Imath_half other);

// unary -
Imath_half Imath_half_neg(const Imath_half* self);
```

The conversion operator should be `to_<type>`:
```c++
// C++
namespace Imath {
class half {
    operator float() const;
} CPPMM_OPAQUEBYTES;
}

// C
float Imath_half_to_float(const Imath_half* self);
```

# Miscellaneous conversion rules

## Functions that return `std::string`

### Option 1
Add an output `char* buffer` and `int len` for the caller to provide pre-allocated storage for the string:
```c++
int OIIO_geterror(char* buffer, int len) {
    std::string err = OIIO::geterror();
    // copies maximum `len` chars to buffer and makes sure last character is
    // always '\0'
    safe_strcpy(buffer, err.c_str(), len);
    // return the actual length of the original string
    return err.size();
}
```
This works well for things like error messages that are expected to be of a "reasonable" length, but what about something that could generate an arbitrarily big string (e.g. PTX generation)?

### Option 2
Create two functions, one that gets the length of the string, and one that copies over the actual data. In this model, the `len` function actually gets the string from C++ and stashes it in a thread_local for later retrieval:

```c++
thread_local std::string err;
int OIIO_geterror_len() {
    err = OIIO::geterror();
    return err.size();
}

int OIIO_geterror(char* buffer, int len) {
    // copies maximum `len` chars to buffer and makes sure last character is
    // always '\0'
    safe_strcpy(buffer, err.c_str(), len);
    // return the actual length of the original string
    return err.size();
}
```
