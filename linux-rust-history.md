## 2021-08-19

```rust

#![no_std]
#![feature(allocator_api, global_asm)]

use kernel::{
    file::File,
    file_operations::FileOperations,
    io_buffer::{IoBufferReader, IoBufferWriter},
    prelude::*,
};
// import more and more

#[derive(Default)]
// I've seen derive(Debug). What is Default?
// see: https://doc.rust-lang.org/std/default/trait.Default.html
// "A trait for giving a type a useful default value."
// But there is no variable to be set with the default value. hmm..
struct RandomFile;

impl FileOperations for RandomFile {
    kernel::declare_file_operations!(read, write, read_iter, write_iter);

    fn read(_this: &Self, file: &File, buf: &mut impl IoBufferWriter, _: u64) -> Result<usize> {
        let total_len = buf.len();
        let mut chunkbuf = [0; 256];

        while !buf.is_empty() {
            let len = chunkbuf.len().min(buf.len());
            let chunk = &mut chunkbuf[0..len];

            if file.is_blocking() {
                kernel::random::getrandom(chunk)?;
            } else {
                kernel::random::getrandom_nonblock(chunk)?;
            }
            buf.write_slice(chunk)?;
        }
        Ok(total_len)
    }

    fn write(_this: &Self, _file: &File, buf: &mut impl IoBufferReader, _: u64) -> Result<usize> {
        let total_len = buf.len();
        let mut chunkbuf = [0; 256];
        while !buf.is_empty() {
            let len = chunkbuf.len().min(buf.len());
            let chunk = &mut chunkbuf[0..len];
            buf.read_slice(chunk)?;
            kernel::random::add_randomness(chunk);
        }
        Ok(total_len)
    }
}

module_misc_device! {
    type: RandomFile,
    name: b"rust_random",
    author: b"Rust for Linux Contributors",
    description: b"Just use /dev/urandom: Now with early-boot safety",
    license: b"GPL v2",
}
```


```rust
#![no_std]
#![feature(allocator_api, global_asm)]

use kernel::pr_cont;
// import pr_cont macro in kernel crate (rust/kernel/print.rs)


use kernel::prelude::*;

module! {
    type: RustPrint,
    name: b"rust_print",
    author: b"Rust for Linux Contributors",
    description: b"Rust printing macros sample",
    license: b"GPL v2",
}

struct RustPrint;

impl KernelModule for RustPrint {
    fn init() -> Result<Self> {
        pr_info!("Rust printing macros sample (init)\n");

        pr_emerg!("Emergency message (level 0) without args\n");
        pr_alert!("Alert message (level 1) without args\n");
        pr_crit!("Critical message (level 2) without args\n");
        pr_err!("Error message (level 3) without args\n");
        pr_warn!("Warning message (level 4) without args\n");
        pr_notice!("Notice message (level 5) without args\n");
        pr_info!("Info message (level 6) without args\n");

        pr_info!("A line that");
        pr_cont!(" is continued");
        // ok, contiguous print.. why??
        
        pr_cont!(" without args\n");

        pr_emerg!("{} message (level {}) with args\n", "Emergency", 0);
        pr_alert!("{} message (level {}) with args\n", "Alert", 1);
        pr_crit!("{} message (level {}) with args\n", "Critical", 2);
        pr_err!("{} message (level {}) with args\n", "Error", 3);
        pr_warn!("{} message (level {}) with args\n", "Warning", 4);
        pr_notice!("{} message (level {}) with args\n", "Notice", 5);
        pr_info!("{} message (level {}) with args\n", "Info", 6);

        pr_info!("A {} that", "line");
        pr_cont!(" is {}", "continued");
        pr_cont!(" with {}\n", "args");

        Ok(RustPrint)
    }
}

impl Drop for RustPrint {
    fn drop(&mut self) {
        pr_info!("Rust printing macros sample (exit)\n");
    }
}
```

```rust
#![no_std]
#![feature(allocator_api, global_asm)]

use kernel::prelude::*;

module! {
    type: RustModuleParameters,
    name: b"rust_module_parameters",
    
    // All string starts with 'b'
    
    author: b"Rust for Linux Contributors",
    description: b"Rust module parameters sample",
    license: b"GPL v2",
    // Just do like this!!!
    params: {
        my_bool: bool {
            default: true,
            permissions: 0,
            description: b"Example of bool",
        },
        my_i32: i32 {
            default: 42,
            permissions: 0o644,
            description: b"Example of i32",
        },
        my_str: str {
            default: b"default str val",
            permissions: 0o644,
            description: b"Example of a string param",
        },
        my_usize: usize {
            default: 42,
            permissions: 0o644,
            description: b"Example of usize",
        },
        my_array: ArrayParam<i32, 3> {
            default: [0, 1],
            permissions: 0,
            description: b"Example of array",
        },
    },
}

struct RustModuleParameters; // MUST implement KernelModule and Drop traits

impl KernelModule for RustModuleParameters {
    fn init() -> Result<Self> {
        pr_info!("Rust module parameters sample (init)\n");

        {
            let lock = THIS_MODULE.kernel_param_lock();
            // type is KParamGuard
            // automatically unlock at the end of this block??
            
            pr_info!("Parameters:\n");
            pr_info!("  my_bool:    {}\n", my_bool.read());
            
            // don't know how my_bool.read() works..
            // Why my_bool.read() without lock but my_i32.read with lock?
            
            
            pr_info!("  my_i32:     {}\n", my_i32.read(&lock));
            pr_info!(
                "  my_str:     {}\n",
                core::str::from_utf8(my_str.read(&lock))?
            );
            pr_info!("  my_usize:   {}\n", my_usize.read(&lock));
            pr_info!("  my_array:   {:?}\n", my_array.read());
        }

        Ok(RustModuleParameters)
    }
}

impl Drop for RustModuleParameters {
    fn drop(&mut self) {
        pr_info!("Rust module parameters sample (exit)\n");
    }
}
```

## 2021-08-17

reference: https://docs.rust-embedded.org/book/intro/no-std.html

```rust
#![no_std] // do not use the standard library such like std crate, but it still link core crate.
#![feature(allocator_api, global_asm)]  // no information found

use kernel::prelude::*;
// rust/kernel/prelude.rs: imports all Rust code.

module! {
    type: RustMinimal,
    name: b"rust_minimal",
    author: b"Rust for Linux Contributors",
    description: b"Rust minimal sample",
    license: b"GPL v2",
}
// see rust/macros/module.rs
// module! macro calls module::module() function.
// module::module() function creates struct ModuleInfo: type, license, name, author(option), description(option), alias(option), params(option)
// type: looks like a name of the structure implementing this module

struct RustMinimal {
    message: String,
}

impl KernelModule for RustMinimal {
// rust/kernel/lib.rs defines KernelModule trait with init function
// init() == module_init of C
    fn init() -> Result<Self> {
        pr_info!("Rust minimal sample (init)\n");
        pr_info!("Am I built-in? {}\n", !cfg!(MODULE));
// cfg!: https://doc.rust-lang.org/reference/conditional-compilation.html
// cfg! returns true if this is a module. So !cfg!() is false if this is a module.

        Ok(RustMinimal {
            message: "on the heap!".try_to_owned()?,
        })
    }
}

impl Drop for RustMinimal {
// Drop should be a destructor of the module.
// But what does the drop() must do?
// self.message is automatically freed AFTER drop().
// Does it have to free memory allocated by init()?
// Or most are freed automatically?
    fn drop(&mut self) {
        pr_info!("My message is {}\n", self.message);
        pr_info!("Rust minimal sample (exit)\n");
    }
}

```

## 2021-08-15
read some example modules
- setup rust-analyzer to read rust core library
  * https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/quick-start.rst#rust-analyzer
  * change rust setting of visual studio code from Rust to rust-analyzer
  * `make rust-analyzer` to create rust-project.json
  * `go to definition` works for core macros and functions
- rust_minimal.rs
- rust_print.rs
  * pr_* functions
- rust_module_parameters.rs
  * macros to define module parameters
  * lock for module parameters setting

## 2021-08-14
booting rust-linux kernel
- copy ubuntu config file /boot/config-5.11.0-25-generic
- enable CONFIG_RUST and CONFIG_SAMPLES_RUST
- build and boot

load sample modules
- `modprobe rust_minimal` : work fine
- build and insmod rust modules
```
make M=samples/rust/ LLVM=1
sudo insmod samples/rust/rust_minimal.ko
```

## 2021-08-13
setup guest os
- install packages for kernel build: build-essential emacs gnome-tweak-tool git libssl-dev libelf-dev
- git clone https://github.com/gurugio/env.git
- rustup
- download github.com/Rust-for-Linux/linux
- apt install clang clang-12 lld lld-12 llvm 
- sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-12 100
- sudo update-alternatives --install /usr/bin/ld.lld ld.lld /usr/bin/ld.lld-12 100
- cp /boot/config~~ .config
- make menuconfig
- enable CONFIG_RUST and CONFIG_SAMPLES_RUST 
- must do `rustup component add rust-src` (https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/quick-start.rst)
- must do `cargo install --locked --version 0.56.0 bindgen`  (https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/quick-start.rst)
- make LLVM=1 -j4
- build rust modules failed: enable CONFIG_DEBUG_INFO and build again

## 2021-08-10
Download https://github.com/Rust-for-Linux/linux

read https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/quick-start.rst

Install build tools
- My rustc version is 1.50.0
- do 'rustup update'
- Now rustc is 1.54.0
- sudo apt-get install libclang-dev
- cargo install --locked --version 0.56.0 bindgen
- make LLVM=1
- clang not found error
- sudo apt-get install clang
- linker 'ld.lld' not found
- sudo apt-get install lld
```
*** Compiler is too old.
***   Your Clang version:    10.0.0
***   Minimum Clang version: 10.0.1
```
- sudo apt-get install clang-12 => no fix
- sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-12 100
- sudo update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-10 100
```
*** Linker is too old.
***   Your LLD version:    10.0.0
***   Minimum LLD version: 10.0.1
```
- sudo apt-get install lld-12
- sudo update-alternatives --install /usr/bin/lld lld /usr/bin/lld-12 100
- ln -s ld.lld-12 ld.lld

Kernel Configuration
make x86_64_defconfig
- 'make menuconfig', then enable CONFIG_RUST and CONFIG_SAMPLES_RUST
- make LLVM=1

QEMU booting
- install: qemu-system-x86_64 -enable-kvm -smp 8 -m 4096M -hda ubuntu.img -cdrom ubuntu-20.04.2.0-desktop-amd64.iso -boot d 
- 1st try: qemu-system-x86_64 -enable-kvm -smp 8 -m 4096M -hda ubuntu.img -boot c
- swap fail?
- I selected automatic installation and it didn't activate swap area.
- I splited partitions manually and created swap area.
- normal booting command: qemu-system-x86_64 -enable-kvm -smp 8 -m 4096M -drive id=d0,file=ubuntu.qcow2,if=none,format=qcow2 -device virtio-blk-pci,drive=d0


TODO:
- install build-tools and build linux-rust.
