Summary
=======

We need to figure out rules for how we map types from C++ to C. There's several considerations that need to be made such as public/private attributes in a class or struct and std library types. The easiest solution would be to have everything go on the heap and have all the types in C forward declare the C++ classes, but that is not going to be the best in terms of performance.

So, there's some rules that we can follow to handle public, private, and standard library types.

Solution
========

We follow the following set of rules:

- If a type contains a std library type, then it has to on the heap.
- If a type does not contain a std library type, but it contains private fields, then it has to be represented in c as an unsigned char array.
- If the type does not contain a std library type, and it only contains public fields, then it can be implementented as a struct in C.

Examples
========

Heap
----

### C++

```cpp
class MyType {
    std::string my_value;
}
```

### C

```c
struct MyType;
```

Private
-------

### C++

```cpp
class MyType {
private:
    int my_value;
}
```

### C

```c
typedef unsigned char MyType[2];
```


Public
------

### C++

```cpp
class MyType {
public:
    int my_value;
}
```

### C

```c
typedef struct {
    int my_value;
}
```
