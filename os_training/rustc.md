# Stable MIR interpreter?

## Rust to Binary flow

[<img src="./doc_pic/compile_flow.png" style="zoom:67%;" />](https://files.solson.me/miri-slides.pdf)

Rust complier(rustc) works on the AST -> ... -> LLVMIR parts. There is a Typed HIR between HIR and MIR in the current version. Note that these IRs are unstable as the compiler version is updated rapidly.

## MIR interpreter

The [interpreter](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/interpret/index.html) is used whenever the compiler must determine the value of an expression to proceed with the compilation. Also know as Compile-Time Function Evaluation (CTFE).

```rust
const fn add_one(x: u32) -> u32 {
    x + 1
}
const FIVE: u32 = add_one(4);
```

To know that `FIVE` is `5`, the compiler does:

1. Generates the MIR for the `add_one(4)` call.
2. Feeds that MIR into its internal MIR interpreter.
3. The interpreter *runs* the MIR and produces the value `5`.
4. The compiler then replaces `FIVE` with the constant value `5` for the rest of the compilation.

Moreover, this interpreter is also used extensively by Miri. Miri overwrites and implements the traits in [`interpret::machine`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_const_eval/interpret/machine/index.html) to keep track of all the information during the MIR interpretation and perform the UB check.

However, the interpreter is unstable. It varies as the complier version changes due to the difference in the MIR generated.

## Stable MIR interpreter

In order to make KernMiri, or Miri, more general to all Rust toolchains, a stable MIR interpreter is crucial. Yet, such an interpreter does not exist.

As I know, there is [Stable-MIR](\text{https://github.com/rust-lang/project-stable-mir) provided by Rust official, but no corresponding interpreter. Stable-MIR stems from the idea that tools for static code analysis, such as Kani, should focus mainly on the analytical method instead of dealing with the rustc information. It sounds reasonable not to have an interpreter. So, one way to achieve our goal is to implement a stable interpreter for Stable-MIR.

Another tool, [Charon](\text{https://github.com/AeneasVerif/charon/tree/main), also provides a way to extract the MIR. It transforms the MIR into LLBC (Low-Level Borrow Calculus) and ULLBC (Unstructured LLBC) formats, which are basically a cleaned-up version of AST and MIR. Kani adopted this tool for its analytic work. Writing an interpreter for ULLBC may be an alternative solution.

## Reference

1. https://files.solson.me/miri-slides.pdf
2. https://files.solson.me/miri-report.pdf
3. https://github.com/endorlabs/MIRAI
4. https://github.com/rust-lang/project-stable-mir
5. https://github.com/AeneasVerif/charon/tree/main
6. https://www.themoonlight.io/zh/review/charon-an-analysis-framework-for-rust
7. https://rustc-dev-guide.rust-lang.org/overview.html
8. https://rustc-dev-guide.rust-lang.org/const-eval/interpret.html

