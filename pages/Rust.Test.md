- [[Rust]] 现在的集成测试逻辑比较慢
- ```rust
  tests/
    foo.rs
    bar.rs
  ```
- 这样结构的 tests 会每次都 link 一遍，导致编译非常慢，所以通常的可以把他们都放到同一个 mod 下面，比如
- ```rust
  tests/
    integration/
      main.rs
      foo.rs
      bar.rs
  ```
- > When a refactoring along these lines was applied to Cargo itself, the effects were substantial (numbers). The time to compile the test suite decreased 3x. The size of on-disk artifacts decreased 5x.
-
- 对集成测试，我们可以
	- ```rust
	  tests/
	    it.rs
	  
	  # Or, for larger crates
	  
	  tests/
	    it/
	      main.rs
	      foo.rs
	      bar.rs
	  ```
- 对内部的一些测试，我们可以
	- ```rust
	  src/
	    lib.rs
	    tests.rs
	    tests/
	      foo.rs
	      bar.rs
	  ```
-
- 尽量避免使用 doctest，非常慢(
-
- `#[test]` 很慢，我们可以使用自己的 Test Harness
	- ```rust
	  [[test]]
	  name = "integration"
	  path = "integration/main.rs"
	  harness = false
	  ```
	- ```shell
	  cargo test integration
	  ```
-
- 参考资料
	- [Delete Cargo Integration Tests](https://matklad.github.io/2021/02/27/delete-cargo-integration-tests.html)
	- [Databend: Further discussion on testing styles](https://github.com/datafuselabs/databend/discussions/3473)
	- [How to Build a Custom Test Harness in Rust](https://www.infinyon.com/blog/2021/04/rust-custom-test-harness/)