Summary
=======
C++ allows for the parameters of a function or method to contain default values.
Rust does not support this.

Default parameters allow for a function/method to provide at once:
* a simple interface with reduced control
* a verbose interface with greater control

There are currently three prevalent patterns in the rust community that solve
this problem.

* Descriptors
* Builders
* Option type directly

It is desirable, that the vfx-rs crates are consistent in this aspect.

Solution
========

For a class that looks like this in c++.

```c++
class Person
{
public:
    Person(const std::string_value & name,
           unsigned int height=50,
           unsigned int weight=70);
};
```

A descriptor might look like this:

```rust

pub mod desc {
    pub struct New<'a> {
        pub name : &'a str,
        pub height : Option<u32>,
        pub weight: Option<u32>,
    }
}

pub struct Person {
    //..
}

impl Person {
    pub fn new(params : desc::New) -> Self {
        // ..
    }
}

fn main {
    let person = Person::new(
        desc::New{
            name : "steve",
            height : Some(34),
            weight : None,
        }
    );
}

```

A builder might look like this:

```rust

pub mod build {
    pub struct New<'a> {
        name : &'a str,
        height : u32,
        weight: u32,
    }

    impl<'a> New<'a> {
        pub fn new(name : &'a str) -> Self {
            Self {
                name,
                height : 50,
                weight: 70,
            }
        }

        pub fn height(mut self, height: u32) -> Self {
            self.height = height;
            self
        }

        pub fn weight(mut self, weight: u32) -> Self {
            self.weight = weight;
            self
        }

        pub fn build(mut self) -> Self { // Not always necessary
            self
        }
    }
}

pub struct Person {
    //..
}

impl Person {
    pub fn new(params : build::New) -> Self {
        // ..
    }
}

fn main {
    use build::New;
    let person = Person::new(
            New::new("steve")
                    .height(32)
                    .build()
    );
}

```

Both examples are clear to read, but as more parameters
are added, the descriptor will become more verbose.
This can be mitigated with additional From and Default trait implementations,
but not entirely, as From traits are type based, and you would not be able
to implement From twice, once for '''height''' and once for '''weight'''.
Additional new methods could be added to the Builder, but then, why not
do that on the original Person struct to begin with.

The builder allows the user to only specify the parameters that are
necessary for them. It also remains source compatible when new parameters are
added.

A final alternative is to use Option type in the method parameters directly.

```rust
impl Person {
    pub fn new(name: &str,
               height: Option<u32>,
               weight: Option<u32>) -> Self {
        // ..
    }
}
```

This is the quickest to implement, but like the descriptor, does not remain
source compatible when additional parameters are added to ```new```.

*No solution yet*.
