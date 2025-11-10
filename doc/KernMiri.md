# KernMiri

For introduction and shims list, please visit https://hackmd.io/bLRtlCH0T9udi9OLGrrXuQ#KernMiri-An-Overview

## Installation

```shell
$ git clone --single-branch -b KernMiri-2024-11-04 git@github.com:billlosw/kern_miri.git kern_miri
$ cd kern_miri
$ rustup install nightly-2024-11-04
$ rustup override set nightly-2024-11-04
$ rustup component add cargo rust-src rustc-dev llvm-tools rustfmt clippy
$ ./miri install

# this should show the same commit hash as the branch in kern_miri.git
$ cargo miri --version

# if no miri is found, try the following.
$ export PATH=$PATH:$HOME/.rustup/toolchains/nightly-2024-11-04-x86_64-unknown-linux-gnu/bin
# or for fish shell
$ set -gx PATH $PATH $HOME/.rustup/toolchains/nightly-2024-11-04-x86_64-unknown-linux-gnu/bin
```

Note that running `./miri toolchain` will raise an error `toolchain xxxx doesn't exist in any channel`. This is because Rust only retains the compiled artifacts from commits within a certain period. Hence, the toolchain must be installed manually..

## How to run

### Run with Asterinas

#### 1\. Clone the Repository

```shell
$ git clone --single-branch -b miri_asterinas \
  https://github.com/asterinas/atc25-artifact-evaluation.git miri_asterinas
$ cd miri_asterinas
```

#### 2\. Apply Source Code Modifications

- Use a global search-and-replace (e.g., in VS Code) to replace all occurrences of:
  * `kern_miri_copy` to `kern_miri_copy_untyped`

- Similarly, replace all occurrences of:
  * `ActionChoice::Miri => todo!()` to `ActionChoice::Miri => return Ok(())`

#### 3\. Run the Miri Interpretation

Execute the following commands from the root directory of the cloned repository (`miri_asterinas`):

```shell
$ rustup override set nightly-2024-11-04
$ make install_osdk
$ mkdir -p test/build && touch test/build/initramfs.cpio.gz
$ cd ostd
$ RUSTFLAGS="-A warnings" \
  MIRIFLAGS="-Zmiri-disable-stacked-borrows -Zmiri-ignore-leaks" cargo osdk miri run
```

This final command will start the interpretation of the Asterinas kernel within the Miri environment.

### Run with other (axplat example)

Basically just add a new function `fn miri_start(_argc: isize, _argv: *const *const u8) -> isize` as the entry point of Miri. For instance,`examples/miri-hello-kernel/src/main.rs`:

```rust
#![no_std]
#![no_main]
extern crate axplat_kernmiri;

#[unsafe(no_mangle)]
fn miri_start(_argc: isize, _argv: *const *const u8) -> isize {
    main(0,0x44000000);
}

#[axplat::main]
fn main(cpu_id: usize, arg: usize) -> ! {
    axplat::console_println!("Hello, ArceOS!");
    axplat::console_println!("cpu_id = {cpu_id}, arg = {arg:#x}");
    for _ in 0..5 {
        axplat::time::busy_wait(axplat::time::TimeValue::from_secs(1));
        axplat::console_println!("{:?} elapsed.", axplat::time::monotonic_time());
    }
    axplat::console_println!("All done, shutting down!");
    axplat::power::system_off();
}

#[cfg(not(test))]
#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    axplat::console_println!("{info}");
    axplat::power::system_off()
}
```

Then run ` RUSTFLAGS="-A warnings" cargo miri run --target x86_64-unknown-none` under directory `/examples/miri-hello-kernel`.

## Updating the toolchain version

Since Miri uses the interpreter built into the rapidly updated Rust compiler, it can only run on the crate or source code that uses the same toolchain version as Miri.

To obtain a specific toolchain version of KernMiri, we can start from the latest implemented version of KernMiri that's earlier than the target one. Then, go to Miri to look for the daily pull request that matches your target toolchain version. For example, if your toolchain is `nightly-2025-02-01`, [Merge pull request#4169from rust-lang/rustup-2025-02-01](https://github.com/rust-lang/miri/commit/44e98633d17ee22c25bbde166a24761a7b6fac2a) is your target branch. Lastly, merge the target branch into KernMiri, fixing all conflicts and compiler errors.

For instance, updating KernMiri with toolchain `nightly-2025-02-01` to  `nightly-2025-05-20`:

```shell
$ cd kern_miri
$ git remote add upstream git@github.com:rust-lang/miri.git
$ git checkout KernMiri-2025-02-01
# https://github.com/rust-lang/miri/commits/master/?since=2025-05-19&until=2025-05-21
$ git merge f9e968e3c69c2aa878ff98ee046b5933d62b6bd2
# solve the conflicts and errors
$ rustup override set nightly-2025-05-20
$ ./miri install
```

## Providing more shims

Since Miri cannot interpret inline assembly, it has to provide shims for users to replace the assembly or other privileged Rust code with that shim.

The get time tick, for example, in riscv64 and x86, can be obtained by reading the register. Yet, KernMiri has no understanding of the assembly. Thus, we have to provide a shim to simulate such an operation.

Precisely, we navigate into `src/shims/foreign_items.rs::emulate_foreign_item_inner()`, and add the following to it:

```rust
"kern_miri_get_ticks" => {
    let [] = this.check_shim(abi, Conv::Rust, link_name, args)?;
    let duration = this.machine.monotonic_clock.now().duration_since(this.machine.monotonic_clock.epoch());
    let ticks = u64::try_from(duration.as_nanos()).map_err(|_| {
        err_unsup_format!("programs running longer than 2^64 nanoseconds are not supported")
    })?;
    this.write_scalar(Scalar::from_u64(ticks), dest)?;
}
```

Then from the user's perspective, replace the assembly with shims:

```rust
unsafe extern "Rust" {
    fn kern_miri_get_ticks() -> u64;
}
fn current_ticks() -> u64 {
    unsafe { kern_miri_get_ticks() }
    // time::read() as u64
}
```

