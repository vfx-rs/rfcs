Summary
=======

In most cases, if we have a function that can fail, then we can use the wrapped library's mechanisms for handling errors. However, if the library provides no C compatible way to handle errors (such as if it only uses exceptions), then we need to use our own error handling mechanisms.

Solution
========

If the library lets us use its error handling mechanisms, then use that in order to make the C interface as similar to the C++ interface as possible. However, if it does not, then we use the following template:

### C++

```cpp
std::thread_local std::string error_string;

int some_function_that_throws() {
    try {
        CPP::some_function_that_throws();
    } catch (const Exception1& e) {
        error_string = e.what();
        return 1;
    } catch (const Exception2& e) {
        error_string = e.what();
        return 2;
    }
    return 0;
}

const char* get_error() {
    return error_string.c_str();
}
```

### C

```c
int some_function_that_throws();

const char* get_error();
```

Examples
========

Status Argument
---------------

### C++

```cpp
Status status;

some_failable_function(&status);

if (!status) {
    ...
}
```

### C

```c
Status status;

some_failable_function(&status);

if (!status_success(&status)) {
    ...
}
```

Get Error
---------

### C++

```cpp
if (!some_failable_function()) {
    get_error();
}
```

### C

```c
if (!some_failable_function()) {
    get_error();
}
```

Exceptions
----------

### C++

```cpp
try {
    some_failable_function();
} catch (const std::exception &err) {
    ...
}
```

### C

```c
if (!some_failable_function()) {
    get_error();
}
```
