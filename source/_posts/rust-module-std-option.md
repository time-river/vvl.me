---
title: Rust Module std::option 1.0.0
date: 2018-04-01 17:21:14
tags: ['language', 'Rust', 'std', 'translation']
---

Module std::option

> 可选的值。

`Option`类型代表了一种可选的值：每个`Option`要么是包含了值的`Some`类型，或者什么都不是的`None`。`Option`类型在Rust代码中很常见，有着大量的使用：

* 初始化值
* 返回函数的值，这个返回值可能没有在整个输入范围内定义（部分功能）
* 返回一个值，否则报告一个`None`，他代表一个简单的错误
* 可选的struct成员
* struct fields that can be loaned or "taken"
* 可选的函数参数
* 为空的指针（nullable pointers）
* 在极端情况下交换值

`Option`常用于模式匹配，对不同的值执行不同的动作，总有针对`None`的情况。

```rust
fn divide(numerator: f64, denominator: f64) -> Option<f64> {
    if denominator == 0.0 {
        None
    } else {
        Some(numerator / denominator)
    }
}

// The return value of the function is an option
let result = divide(2.0, 3.0);

// Pattern match to retrieve the value
match result {
    // The division was valid
    Some(x) => println!("Resukt: {}", x),
    // The division was invalid
    None    => println!("Cannot divide by 0"),
}
```

> Note：
>   Question：关于`xxx }`与`return xxx };`的区别。
>   Answer：可以认为没有区别吧，详细解释如下：
>       [Why is using return as the last statement in a function considered bad style?](https://stackoverflow.com/questions/27961879/why-is-using-return-as-the-last-statement-in-a-function-considered-bad-style)
>       [Why isn't the syntax of return statements explicit?](https://www.reddit.com/r/rust/comments/2v82ag/why_isnt_the_syntax_of_return_statements_explicit/)

## Options and pointers("nullable" pointers)

Rust的指针类型一定指向一个有效的位置；没有空指针。相反，Rust有*optional*指针，像`Option<Box<T>>`一样。

下面的例子使用`Option`来创建一个`i32`类型的option box。注意，为了先使用内置的`i32`类型值，`check_optional`函数需要使用模式匹配来决定box是否有一个值（比如，它是`Some(...)`）或者值不存在（`None`）。

```rust
let optional = None;
check_optional(optional);

let optional = Some(Box::new(9000));
check_optional(optional);

fn check_optional(optional: Option<Box<i32>>) {
    match optional {
        Some(ref p) => println!("has value {}", p),
        None => println!("has no value"),
    }
}
```

使用`Option`来创建安全的nullable pointers很常见，Rust为`Option<Box<T>>`表示成一个单独的指针做了优化。无论是哪种指针类型，Rust中的Optional pointers的效率都很好。

## Examples

`Option`基本的模式匹配：

```rust
let msg = Some("howdy");

// Take a reference to the contained string
if let Some(ref m) = msg {
    println!("{}",*m);
}

// Remove the contained string, destroying the Option
let unwrapped_msg = msg.unwrap_or("default message");
```

循环之前对`None`结果初始化：

```rust
enum Kingdom { Plant(u32, &'static str), Animal(u32, &'static str) }

// A list of data to search through.
let all_the_big_things = [
    Kingdom::Plant(250, "redwood"),
    Kingdom::Plant(230, "noble fir"),
    Kingdom::Plant(229, "sugar pine"),
    Kingdom::Animal(25, "blue whale"),
    Kingdom::Animal(19, "fin whale"),
    Kingdom::Animal(25, "north pacific right whale"),
];

// We're going to search for the name of the biggest animal,
// but to start with we've just got `None`.
let mut name_of_biggest_animal = None;
let mut size_of_biggest_animal = 0;
for big_thing in &all_the_big_things {
    match *big_thing {
        Kingdom::Animal(size, name) if size > size_of_biggest_animal => {
            // Now we've found the name of some big animal
            size_of_biggest_animal = size;
            name_of_biggest_animal = Some(name);
        }
        Kingdom::Animal(..) | Kingdom::Plant(..) => ()
    }
}

match name_of_biggest_animal {
    Some(name) => println!("the biggest animal is {}", name),
    None => println!("there are no animals:（"),
}
```

## Structs

...

## Enums

...

## Orginal

[Module std::option 1.0.0](https://doc.rust-lang.org/std/option/)