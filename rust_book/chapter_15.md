# Notes on Smart Pointers: Chapter 15 of The Rust Book

A *pointer* is variable containing an address in memory

A reference is the most common type of pointer in Rust and are indicated by the & symbol
They borrow the value they point to

Smart pointers have additional metadata and capabilities

While a reference points to data on the stack, the Smart pointer can also point to data on the heap

Smart pointers also oten *own* the data they point to unlike a reference

A *reference counting* smart pointer enables you to allow data to have multiple owners by keeping track of the number of owners and, when no owners remain, cleaning up the data

`String` and `Vec<T>` are examples of Smart pointers we have already seen; they own some memory and allow you to manipulate it

The metadata of a String includes capacity and has the ability to ensure its data will always be valid UTF-8

Smart pointers are usually implemented using structs, but require implementing Deref and Drop traits

`Deref` allows an instance of a smart pointer struct to behave like a reference so you can write your code to work with either references or smart pointers

`Drop` allows you to customize the code that's run when an instance of the spart pointer goes out of scope

The most basic Smart pointers in the standard library are:
* Box<T>, for allocating values on the heap
* Rc<T>, a rererence counting type that enables multiple ownership
* Ref<T> and RefMut<T>, accessed through RefCell<T>, a type that enforces the borrowing rules at runtime instead of compile time

## Using Box<T> to Point to Data on the Heap
Box<T> allows you to store data on the heap rather than the stack; the stack only contains the pointer to the heap data

Boxes don't have performance overhead

Useful situations for Boxes:
* When you have a type whose size can’t be known at compile time and you want to use a value of that type in a context that requires an exact size
* When you have a large amount of data and you want to transfer ownership but ensure the data won’t be copied when you do so
* When you want to own a value and you care only that it’s a type that implements a particular trait rather than being of a specific type

In the second situation, we can improve performance in this situation by storing the large amount of data on the heap in a box

```
fn main() {
    let b = Box::new(5);
    println!("b = {b}");
}
```

Deallocation of both the pointer and the heap daya occur when the box goes out of scope, in this case at the end of `main`

### Enableing Recursive Types with Boxes

```
enum List {
    Cons(i32, List),
    Nil,
}
```

```
use crate::List::{Cons, Nil};
fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```

This will result in an error because this type "has infinite size"

```
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

When allocating space on the heap for this custom type, Rust determines this as which variants requires the most space

The error from the List example above is that we should "insert some indirection to make `List` representable

This means to store the value indirectly by storing a pointer to the value instead:

```
enum List {
    Cons(i32, Box<List>),
    Nil,
}
use crate::List::{Cons, Nil};
fn main() {
    let list = Cons(
        1,
        Box::new(Cons(
            2,
            Box::new(Cons(
                3,
                Box::new(Nil)
            ))
        ))
    );
}
```

By using a box, we've broken the infinite, recursive chain, so the compiler can figure out the size it needs to store a `List` value

The `Box<T>` type is a smart pointer because it implements the `Deref` trait, which allows `Box<T>` values to be treated like references. When a `Box<T>` value goes out of scope, the heap data that the box is pointing to is cleaned up as well because of the `Drop` trait implementation.

## Treating Smart Pointers Like Regular References with Deref

Implementing the `Deref` trait allows you to customize the behavior of the *dereference operator \** 

By implementing `Deref` in such a way that a smart pointer can be treated like a regular reference, you can write code that operates on references and use that code with smart pointers too

### Defining our Own Smart Pointer
```
struct myBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
```

In order to dereference a `let y = MyBox::new(x);`, we need to implement the `Deref` trait:

```
use std::ops::Deref;
impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

0 accesses the first value ina tuple struct

Now the following will work:
```
fn main() {
    let x = 5;
    let y = MyBox::new(x);
    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

*** Note that this example does not actually allocate any memory on the heap; this is simply to demonstrate deferencing

### Implicit Deref Coercions with Funcitons and Methods
