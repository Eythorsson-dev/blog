---
title: Rust Macros
date: 2024-09-15
---

___Rust macros. What are they?___
Rust macros, are a way of writing code that writes itself. It enables code reuse, and simplifies boilerplate


___How do they work?___
We define a macro using `macro-rules! <Name>`, and specify its name. Next comes the body. The body can be though of as a match expression (with a differrent syntax).

```rust
macro_rules! my_println {
    ( $( ($x:expr, $y:expr) ),* ) => {
        {
            $(
                println!("Hello World {:?}", $y);
                println!("Hello World {:?}", $x);
            )*
        }
    };

    ( $( $x:expr ),* ) => {
        {
            $(
                println!("Hello World {:?}", $x);
            )*
        }
    };
}

#[test]
fn test_my_vec() {
    my_println![1, 2, 3];
    // Prints:
    // Hello World 1
    // Hello World 2
    // Hello World 3

    my_println![("Foo", 2), ("Bar", 4)];
    // Prints:
    // Hello World 2
    // Hello World "Foo"
    // Hello World 4
    // Hello World "Bar"
}
```

In the above macro, the `( $( $x:expr ),* )` line is the expression to match. The initial `$(` is he opening block and the `),*` means that we could have one or more instances of the the input parameter `$x:expr`. Notice that this macro contains an multiple match bodies. this causes `my_println![("Foo", 2), ("Bar", 4)]` to be handled diffrently than `my_println![1, 2, 3]`. Without the `( $( ($x:expr, $y:expr) ),* ) ` mactch expression. The `my_println![("Foo", 2), ("Bar", 4)]` would print as
```
Hello World ("Foo", 2)
Hello World ("Bar", 4)
```

___How can we create matches___
https://doc.rust-lang.org/rust-by-example/macros/designators.html

