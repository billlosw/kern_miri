# KernMiri

For introduction, please visit https://hackmd.io/bLRtlCH0T9udi9OLGrrXuQ#KernMiri-An-Overview

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

The crate or source code must use the same toolchain version as KernMiri. Take axplat as an example.

TODO!

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
# solve the damn conflicts and errors
$ rustup override set nightly-2025-05-20
$ ./miri install
```

