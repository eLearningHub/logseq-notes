title:: Rust/std/Try

- [Try](https://doc.rust-lang.org/std/ops/trait.Try.html) 是 [[Rust]] 在 `try_trait_v2` feature 中引入的新 ops，旨在支持 `?` 操作符和 `try {}` block。
	- Tracking Issue: [Tracking Issue for try_trait_v2, A new design for the ? desugaring](https://github.com/rust-lang/rust/issues/84277)
-
- 现在 Rust 中支持通过 `?` 来自动返回 `Result<T, E>` 中的 `Err` 分支，但是 `Ok` 分支