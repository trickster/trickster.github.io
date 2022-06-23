+++
title = "Error handling in Rust"
date = 2021-06-11T08:10:55+09:00
type = "post"
description = "Notes on error handling in Rust"
in_search_index = true
[taxonomies]
tags = ["Dev", "Rust"]
+++

## Creating new error types

Let's wrap std library errors with our Errors

```rust
use std::fmt; 
use std::fmt::Debug; 
use std::num::{ParseIntError, ParseFloatError}; 

#[derive(Debug)] 
enum MyErrors { 
    Error1, 
    Error2 
}
```

We also need to implement fmt::Display for our Errors.

```rust
impl fmt::Display for MyErrors { 
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result { 
        match *self { 
            MyErrors::Error1 => write!(f, "my first error"),
            MyErrors::Error2 => write!(f, "my second error"), 
        } 
    } 
} 
```

Let's implement our own function to test this [1]

```rust
fn test_error1() -> Result<(), MyErrors> { 
    let _ = "a".parse::<i32>()?; 
    let _ = "a".parse::<f64>()?; 
    Ok(()) 
}
fn test_error2() -> Result<(), MyErrors> { 
    let _ = "1".parse::<i32>()?; 
    let _ = "a".parse::<f64>()?; 
    Ok(()) 
}
```

The above one raises error, because Rust doesn't know how to convert ParseIntError to MyErrors. We need to implement them using From trait.

```rust
impl From<ParseIntError> for MyErrors { 
    fn from(err: ParseIntError) -> MyErrors { 
        MyErrors::Error1 
    } 
}
impl From<ParseFloatError> for MyErrors { 
    fn from(err: ParseFloatError) -> MyErrors { 
        MyErrors::Error2
    } 
}
```

You can also avoid the above step and convert it explicitly using

```rust
let _ = "a".parse::<i32>().map_err(|_| MyErrors::Error1)?;
```

But, implementing From once and using it everywhere is less verbose.

We can use these in our main function which returns our error type on failure.

```rust
fn main() -> Result<(), MyErrors> { 
    //test_error1()?;     
    if let Err(e) = test_error1() { 
        println!("Error message: {}", e) 
    }
    if let Err(e) = test_error2() { 
        println!("Error message: {}", e) 
    } 
    Ok(()) 
}
```

If we want to use `std::error::Error` in main, we need to implement it for `MyErrors` using

```rust
impl std::error::Error for MyErrors{} 
```

Then we can use

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
   ...
}
```

Full working example using everything above. Remember if you don't want infallible main, you can just print the error message in main if any, otherwise use `?` operator.

## Wrapping underlying errors

We can create our error type that can directly wrap underlying error and print backtrace.

```rust
#[derive(Debug)] 
enum MyErrors { 
    Error1(ParseIntError), 
    Error2(ParseFloatError) 
}
// Implement Display
impl fmt::Display for MyErrors { 
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result { 
        match *self { 
             MyErrors::Error1(..) => write!(f, "my first error"), // wrap ParseIntError 
             MyErrors::Error2(..) => write!(f, "my second error"), // wrap ParseFloatError 
        } 
    } 
}
```

We need to implement From for converting error types from one to another

```rust
// convert ParseIntError to MyErrors::Error1 
impl From<ParseIntError> for MyErrors { 
    fn from(err: ParseIntError) -> MyErrors { 
        MyErrors::Error1(err) 
    } 
} 

// convert ParseFloatError to MyErrors::Error1 
impl From<ParseFloatError> for MyErrors { 
    fn from(err: ParseFloatError) -> MyErrors { 
        MyErrors::Error2(err) 
    } 
}
```

We can use the same test functions as above [1] and we need backtrace. For backtrace, instead of empty impl `std::error::Error` for our error type, we can implement source

```rust
impl std::error::Error for MyErrors { 
    // if you want source of the error ---- 
    // In our case display error message related to 
    // ParseIntError or ParseFloatError 
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> { 
        match *self { 
            MyErrors::Error1(ref e) => Some(e), 
            MyErrors::Error2(ref e) => Some(e), 
        } 
    } 
}
```

and we have our main function like this

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> { 
    // test_error1()?; 
    if let Err(e) = test_error1() { 
         println!("Error message: {}", e); 
         if let Some(source) = e.source() { 
            println!(" Caused by: {}", source); 
         } 
     } 
     Ok(())
}
```

We get this output

```sh
Error message: my first error
   Caused by: invalid digit found in string
```

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=a7d10bf89ae970fca143d494d11c0e95) for above -- with boilerplate

To create our own error type, we implemented, From conversions, std::error::Error and it's methods. In order to avoid all these, we can use a trait called `thiserror`

```rust
#[derive(thiserror::Error, Debug)] 
enum MyErrors { 
    #[error("my first error")] 
    Error1(#[from] ParseIntError), 
    #[error("my second error")] 
    Error2(#[from] ParseFloatError) 
}
```

The above implements custom error type with our message, from conversions and backtrace.

[Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e50690f48555da81474b98ad4b0d98dd) for above -- using `thiserror`

Both of these, produce the same output. Using thiserror reduced the code size by half in our example
