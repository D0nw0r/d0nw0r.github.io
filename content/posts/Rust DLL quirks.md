+++
date = '2025-11-03T15:52:54+02:00'
draft = false
title = 'Rust DLL quirks'
subtitle = 'Exploring exported functions on Rust DLLs'
+++
<!--more-->

# Rust DLL Export Quirks

Another blog post about DLLs? Well, it wasn't planned at all! But let's get into it.

When exporting functions on a Rust DLL, the functions it calls are also exported and show on the PE exports.

Example:

```Rust
mod proto;
mod func;

#[macro_use]
extern crate litcrypt;
use_litcrypt!();
 

#[unsafe(no_mangle)]
pub extern "C" fn lib_main() {
    proto::Pick();
}
```

```Rust
use crate::func;
use std::fs::File;
use std::io::Write;

#[unsafe(no_mangle)]
pub extern "system" fn Pick() {

    println!("{}", lc!("Calling public hello fn from func.rs"));
    func::hello();
    println!("{}", lc!("Calling private hello fn from proto.rs"));
    hello();
    println!("{}", lc!("Calling public helper fn from func.rs"));
    func::helper();
    // Modify file writing to include counter
    write_to_file();

}

fn write_to_file() {
    let mut file = File::create("output.txt").expect("Unable to create file");
    file.write_all(b"Hello, world!").expect("Unable to write data");

}

fn hello() {
    println!("Hello, world!");
}
```

The DLL exported function appears to be lib_main, however all the other functions called from it also show:
<br>
<br>
<img src="/images/before.png" alt="library usage" style="max-width: 400px; border-radius: 8px;">
<br>



To avoid unnecessary exports, can put all the implant logic into a Rust Library lib.rs file, and mark that entry as `extern "system"`.

```Rust
// lib/src/lib.rs

use windows_sys::Win32::{
    Foundation::HMODULE,
};

  

pub type BOOL = i32;
 

#[unsafe(no_mangle)]
#[allow(non_snake_case)]
pub extern "system" fn DllMain(
    _hinst_dll: HMODULE,
    #[allow(unused)]
    call_reason: u32,
    _lpv_reserved: *mut std::ffi::c_void,
) -> BOOL {
    1
}

#[no_mangle] 
pub extern "C" fn run_me() {
    run_from_lib();
}
```

Inside that function all the other calls **coming from other modules** will not be exported.

```Rust
// lib/src/other.rs
mod other;
use::crate::{other::other_call}

pub fn run_from_lib()  {
    debug_println!("[*] Implant starting");
    other_call();
}
```


`run_me` calls `run_from_lib` only, `other_call` does not appear on the exported functions.
<br>
<br>
<img src="/images/after.png" alt="library usage" style="max-width: 400px; border-radius: 8px;">
<br>


The two calls here are only from the DLL exported function and the **first** function from `lib.rs` which is `run_from_lib`.

Unlike the previous example where subsequent calls inside the `Pick` function (like `hello`) showed as exported too.


Mentions: https://github.com/Teach2Breach/rust_template - I have used his template to showcase the first example of exported functions.
Highly recommend his book!