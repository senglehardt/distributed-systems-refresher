`Box<dyn std::error::Error>` is used for several key reasons:

**1. Trait objects require indirection**
- `dyn std::error::Error` is a trait object with an unknown size at compile time (dynamically sized type)
- Rust requires all return types to have a known size, so you need `Box` (heap allocation) to create a pointer with a fixed size

**2. Return different error types**
- Different operations produce different error types (`std::io::Error`, `ParseIntError`, etc.)
- `Box<dyn Error>` allows you to return ANY error type that implements the `Error` trait from the same function
- Without it, you'd need to define a custom enum or use a concrete error type

**3. Convenient with `?` operator**
- The `?` operator automatically converts most errors into `Box<dyn Error>`
- In your code, `handle.join()?` might fail, and `?` converts that error seamlessly

**Example without Box:**
```rust
// This won't compile - unknown size
fn main() -> Result<(), dyn std::error::Error> { ... }
```

**Alternative approaches:**
- Use `anyhow::Result` or `eyre::Result` (cleaner syntax)
- Define a custom error enum if you need more control
- Use concrete error types like `std::io::Error` if you only have one error type

The "Box" is essentially paying a small heap allocation cost to get type flexibility and proper sizing.