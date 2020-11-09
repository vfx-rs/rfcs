# Summary

Lots of libraries use allocating standard library types in their API and we need a method for dealing with:
- `std::vector`
- `std::string`

The simplest way of dealing with these is to just copy the types entirely to C equivalents in the wrapper functions. This is the method currently proposed for handling strings. For example:
```c++
int OIIO_ImageInput_geterror(const OIIO_ImageInput* self, char* _result_buffer_ptr, int _result_buffer_len) {
    const std::string result = to_cpp(self)->geterror();
    safe_strcpy(_result_buffer_ptr, result, _result_buffer_len);
    return result.size();
}
```
In this model, the caller allocates the memory and passes it in to the wrapper. The wrapper then copies from the string returned from C++ to the provided buffer, making sure not to exceed the user-provided length.

This works fine for short arrays, such as error messages where we expect to be able to stack-allocate the buffer and adjust the program if we find it's not large enough, or allocate a reasonably large one once and reuse it throughout the program, but strings and vectors could really be any size. We need a different model.

The crux of the issue is that `vector` and `string` allocate and manage their memory using RAII and provide no facility for releasing ownership of that memory.

In the examples below we'll just use `std::vector`. There are three cases we need to consider:

```c++
namespace foo {
std::vector<float> returns();
void as_mut_param(std::vector<float>& vec);
void as_const_param(const std::vector<float>& vec);
}
```

# Possible solutions

## 1. Two-phase copy

This is a modifcation of the error-message method above. For returning, we make two functions, say `len` and `get`. `len` is the function that actually gets the vector, then stashes it in a thread_local and returns its size, giving the caller an opportunity to allocate memory of the required size which it can pass to `get`, which will copy from the stashed vector:

```c++
int foo_returns_len() {
    thread_local _stash = foo::returns();
    return _stash.size();
}

void foo_returns_get(float* buffer, int len) {
    memcpy(buffer, _stash.data(), sizeof(float) * min(_stash.size(), len));
    _stash = {};
}
```

`const_param` just copies into a temporary vector:
```c++
void foo_as_const_param(float* buffer, int len) {
    std::vector tmp{buffer, len};
    foo::as_const_param(tmp);
}
```

`mut_param` is trickier since we potentially need to both pass data and return it. Easiest is probably just not to support passing data at all, in which case it reduces to the same as `returns`.

### Pros
- It's guaranteed to work and is safe
- Caller is responsible for all allocations

### Cons
- Slow
- Slightly ugly API for the `returns` case. `len` actually getting the vector is surprising for callers.
- Works for POD elements only (no vector<string>)

## 2. Binding-side storage

In this model we stash the vector in a global (thread_local) map, indexed by its data pointer so it can be found and deleted later:

```c++
thread_local std::map<void*, std::any> _storage;

foo_returns(float** buffer, int* len) {
    std::vector<float> vec = foo::returns();
    *buffer = vec.data();
    *len = vec.size();
    _storage[vec.data()] = std::move(vec);
}

void foo_delete_storage(void* buffer) {
    _storage.erase(buffer);
}
```

`const_param` has to invoke copies, same as method 1 since we have no way of constructing a vector directly. Same goes for `mut_param`, and even though we might be tempted to reuse an existing buffer from a previous call to `foo_returns()`, that would be very bad in the case of the vector reallocating.

### Pros
- The `returns` case is nice

### Cons
- Still copies in the parameter case
- Works for POD elements only (no vector<string>)

## 3. opaquebytes wrapper type

In this method we create an opaquebytes wrapper for a vector in order to pass between C and C++ and implement a custom interface over the held vector.

```c++

// C++
using VecVariant = std::variant<
    std::vector<int>,
    std::vector<float>,
    std::vector<string>,
>;

class VectorWrapper {
    VecVariant _vec;

public:
    VecVariant(){}

    <template typename T>
    VecVariant(std::vector<T>&& v) : _vec(std::move(v)) {}

    <template typename T>
    bool get(int i, T* value) const {
        if (auto* pvec = _vec.get_if<std::vector<T>>()) {
            *value = (*pvec)[i];
            return true;
        } else {
            return false;
        }
    }

    int size() const {
        std::visit([](auto vec) -> size_t {
            return vec.size();
        }, _vec);
    }

};

void foo_returns(cppmm_VecWrapper* wrapper) {
    std::vector<float> vec = foo::returns();
    // placement new move into the caller-allocated memory so as not to call
    // the destructor when this function returns.
    // I *think* this is safe.
    new (wrapper) VecWrapper(std::move(vec));
}

// C
void main(void) {
    VecWrapper vec_wrapper;
    foo_returns(&vec_wrapper);

    float element;
    cppmm_VecWrapper_get_float(&vec_wrapper, 0, &element)

    printf("size is %d and first element is %f\n",
        cppmm_VecWrapper_size(&vec_wrapper), element);

    cppmm_VecWrapper_destruct(&vec_wrapper);
}

```

### Pros
- Zero copy
- Can handle std::string elements as well (with appropriate functions on the wrapper)
- Can ensure move-only in the Rust wrapper and represent as a slice

### Cons
- Adding new types just for the binding layer
- Unsafe if wrapper object is copied in C then passed by value back to C++
- List of types must be statically known and monomorphized in the binding layer (might be able to auto-generate this)
